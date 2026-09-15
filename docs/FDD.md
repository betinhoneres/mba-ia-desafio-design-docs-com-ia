# FDD — Sistema de Webhooks de Notificação de Pedidos

## Status

Pronto

---

# 1. Contexto e Motivação Técnica

Atualmente clientes B2B realizam polling contínuo no endpoint de pedidos para identificar mudanças de status.

Esse modelo gera:

- Carga desnecessária na API.
- Aumento do consumo de recursos computacionais.
- Latência na propagação de mudanças de status.
- Complexidade para integração dos clientes.

A feature introduz um mecanismo de Webhooks Outbound para notificar clientes automaticamente sempre que houver alteração no status de pedidos.

As decisões arquiteturais aprovadas definiram:

- Transactional Outbox.
- Worker dedicado.
- Polling via banco.
- Entrega At-Least-Once.
- HMAC-SHA256.
- Dead Letter Queue.

Este documento descreve como implementar tecnicamente a solução.

---

# 2. Objetivos Técnicos

## Objetivos Primários

- Garantir que toda mudança de status gere um evento persistido.
- Garantir desacoplamento entre alteração do pedido e entrega HTTP.
- Garantir entrega resiliente utilizando retry com backoff.
- Garantir rastreabilidade completa dos eventos.
- Fornecer mecanismos de auditoria e replay.

## Objetivos Secundários

- Reutilizar infraestrutura existente.
- Não introduzir Redis ou Kafka.
- Seguir padrões atuais da aplicação.
- Minimizar impacto no domínio de pedidos.

---

# 3. Escopo e Exclusões

## Escopo

### Configuração de Webhooks

- Cadastro
- Atualização
- Remoção
- Listagem

### Entrega de Eventos

- Geração de eventos
- Persistência na Outbox
- Processamento assíncrono
- Retry
- DLQ
- Replay administrativo

### Segurança

- Assinatura HMAC-SHA256
- HTTPS obrigatório
- Secret exclusiva por webhook
- Rotação de secret

### Observabilidade

- Logs
- Métricas
- Histórico de entregas

---

## Exclusões

Fora de escopo desta implementação:

- Dashboard frontend.
- Notificação por e-mail ao cliente.
- Rate limiting de saída.
- Múltiplos workers.
- Ordering global.
- Redis Streams.
- Kafka.
- Exactly-once delivery.

---

# 4. Integração com o Sistema Existente

## src/modules/orders/order.service.ts

### Alteração

O método:

```ts
changeStatus(...)
```

deverá ser estendido para registrar eventos na Outbox dentro da mesma transação atual.

Fluxo:

```text
update order
insert order_status_history
update stock
insert webhook_outbox
commit
```

Nova chamada:

```ts
publishWebhookEvent(
  tx,
  order,
  fromStatus,
  toStatus
)
```

---

## src/server.ts

### Alteração

Nenhuma alteração funcional.

O módulo de webhooks deverá registrar suas rotas utilizando o mesmo padrão dos demais módulos.

Exemplo:

```ts
app.use('/webhooks', webhookRoutes);
```

---

## src/modules/shared/errors/AppError.ts

### Alteração

Nenhuma.

A feature deverá reutilizar o padrão existente baseado em:

```ts
new AppError(...)
```

Todos os erros deverão seguir o padrão:

```text
WEBHOOK_*
```

---

## src/modules/shared/middlewares/error.middleware.ts

### Alteração

Nenhuma.

Os novos erros do módulo devem ser compatíveis com o middleware existente.

---

## src/modules/shared/logger.ts

### Alteração

Nenhuma.

Todo logging deverá utilizar Pino através da infraestrutura já existente.

---

## Novo Arquivo

```text
src/worker.ts
```

Responsável por inicializar o loop de processamento da Outbox.

---

## Novo Módulo

```text
src/modules/webhooks/
```

Estrutura proposta:

```text
src/modules/webhooks
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
├── webhook.schemas.ts
├── webhook.routes.ts
├── webhook.processor.ts
├── webhook.outbox.ts
├── webhook.dlq.ts
└── errors/
```

---

# 5. Fluxos Detalhados

## Fluxo 1 — Criação do Evento na Outbox

### Trigger

Mudança de status do pedido.

### Sequência

```text
OrderService.changeStatus
            ↓
Validação
            ↓
Transaction Begin
            ↓
Update Order
            ↓
Insert History
            ↓
Update Stock
            ↓
publishWebhookEvent
            ↓
Insert webhook_outbox
            ↓
Commit
```

### Regras

- Participa da mesma transação.
- Rollback caso a inserção falhe.
- Payload é armazenado como snapshot.

---

## Fluxo 2 — Processamento pelo Worker

### Loop

```text
while(true)
```

### Sequência

```text
Buscar eventos pendentes
        ↓
Ordenar por created_at
        ↓
Executar entrega
        ↓
Sucesso?
      /     \
    SIM      NÃO
     ↓        ↓
ENTREGUE   RETRY
```

### Frequência

```text
2 segundos
```

---

## Fluxo 3 — Retry

### Política

| Tentativa | Espera |
|------------|---------|
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

### Fluxo

```text
Falha
 ↓
Incrementa retry_count
 ↓
Calcula next_attempt_at
 ↓
Atualiza evento
```

---

## Fluxo 4 — Dead Letter Queue

### Trigger

```text
retry_count > 5
```

### Fluxo

```text
Evento falhou
        ↓
Inserir webhook_dead_letter
        ↓
Marcar outbox como FAILED
```

---

## Fluxo 5 — Replay

### Endpoint

```http
POST /admin/webhooks/dead-letter/:id/replay
```

### Fluxo

```text
ADMIN
 ↓
Requisição
 ↓
Busca DLQ
 ↓
Cria novo evento na Outbox
 ↓
Marca replay realizado
 ↓
Auditoria
```

---

# 6. Contratos Públicos

## POST /webhooks

### Request

```json
{
  "customer_id": "cus_123",
  "url": "https://customer.com/webhook",
  "events": [
    "SHIPPED",
    "DELIVERED"
  ]
}
```

### Response 201

```json
{
  "id": "wh_123",
  "secret": "generated_secret",
  "url": "https://customer.com/webhook",
  "events": [
    "SHIPPED",
    "DELIVERED"
  ]
}
```

### Status Codes

```text
201 Created
400 Bad Request
401 Unauthorized
409 Conflict
```

---

## PATCH /webhooks/:id

### Request

```json
{
  "url": "https://novo-endpoint.com/webhook",
  "events": [
    "DELIVERED"
  ]
}
```

### Response 200

```json
{
  "id": "wh_123",
  "updated": true
}
```

### Status Codes

```text
200 OK
400 Bad Request
404 Not Found
```

---

## GET /webhooks

### Response 200

```json
{
  "data": [
    {
      "id": "wh_123",
      "url": "https://customer.com/webhook",
      "events": ["SHIPPED"],
      "active": true
    }
  ]
}
```

### Status Codes

```text
200 OK
401 Unauthorized
```

---

## DELETE /webhooks/:id

### Response 204

```json
{}
```

### Status Codes

```text
204 No Content
404 Not Found
```

---

## GET /webhooks/:id/deliveries

### Response 200

```json
{
  "data": [
    {
      "event_id": "evt_123",
      "status": "SUCCESS",
      "response_code": 200,
      "response_time_ms": 145
    }
  ]
}
```

### Status Codes

```text
200 OK
404 Not Found
```

---

## POST /admin/webhooks/dead-letter/:id/replay

### Response 202

```json
{
  "replayed": true
}
```

### Status Codes

```text
202 Accepted
401 Unauthorized
403 Forbidden
404 Not Found
```

---

# 7. Contrato de Entrega do Webhook

## Headers

```http
Content-Type: application/json
X-Event-Id: evt_uuid
X-Signature: generated_hmac
X-Timestamp: 2026-09-14T20:00:00Z
X-Webhook-Id: wh_uuid
```

---

## Payload

```json
{
  "event_id": "evt_uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-14T20:00:00Z",
  "order_id": "ord_123",
  "order_number": "ORD-1001",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "cus_123",
  "total_cents": 15000
}
```

---

# 8. Matriz de Erros

| Código | Descrição | HTTP |
|----------|-------------|------|
| WEBHOOK_NOT_FOUND | Webhook não encontrado | 404 |
| WEBHOOK_INVALID_URL | URL inválida | 400 |
| WEBHOOK_SECRET_REQUIRED | Secret obrigatória | 400 |
| WEBHOOK_INACTIVE | Webhook desativado | 400 |
| WEBHOOK_DELIVERY_FAILED | Falha na entrega | 500 |
| WEBHOOK_SIGNATURE_ERROR | Erro ao gerar assinatura | 500 |
| WEBHOOK_PAYLOAD_TOO_LARGE | Payload excede limite | 400 |
| WEBHOOK_REPLAY_NOT_ALLOWED | Replay não permitido | 403 |
| WEBHOOK_DLQ_NOT_FOUND | Evento DLQ não encontrado | 404 |
| WEBHOOK_TIMEOUT | Timeout na entrega | 504 |

---

# 9. Estratégias de Resiliência

## Timeout

```text
10 segundos
```

Falhas por timeout seguem fluxo de retry.

---

## Retry

Backoff exponencial:

```text
1m
5m
30m
2h
12h
```

---

## Entrega

Semântica:

```text
At-Least-Once
```

Duplicidades são permitidas.

---

## Identificação

Todo evento possui:

```text
event_id
```

Clientes devem realizar deduplicação.

---

## Fallback

Após a última tentativa:

```text
DLQ
```

---

# 10. Observabilidade

## Métricas

### Counters

```text
webhook_events_created_total

webhook_delivery_success_total

webhook_delivery_failed_total

webhook_retry_total

webhook_dlq_total

webhook_replay_total
```

---

### Gauges

```text
webhook_outbox_pending

webhook_dlq_pending
```

---

### Histograms

```text
webhook_delivery_duration_ms

webhook_processing_duration_ms
```

---

## Logs

Todos os logs devem utilizar Pino.

### Eventos

```text
WEBHOOK_CREATED

WEBHOOK_DELIVERED

WEBHOOK_FAILED

WEBHOOK_RETRY

WEBHOOK_MOVED_TO_DLQ

WEBHOOK_REPLAY
```

---

## Tracing

Adicionar os seguintes atributos:

```text
event_id
webhook_id
customer_id
order_id
```

Objetivo:

```text
Pedido
 ↓
Evento
 ↓
Entrega
 ↓
Replay
```

Totalmente rastreável.

---

# 11. Dependências e Compatibilidade

## Dependências

```text
MySQL
Prisma
Node.js
Pino
Zod
JWT
```

---

## Compatibilidade

Reutilizar obrigatoriamente:

```text
AppError
Error Middleware
Prisma Client
Pino
Padrão de módulos
```

---

# 12. Critérios de Aceite Técnicos

## Outbox

- [ ] Evento criado dentro da mesma transação.
- [ ] Rollback ocorre se insert da Outbox falhar.

---

## Worker

- [ ] Executa em processo separado.
- [ ] Polling ocorre a cada 2 segundos.
- [ ] Processa apenas eventos elegíveis.

---

## Segurança

- [ ] Apenas HTTPS permitido.
- [ ] HMAC-SHA256 válido.
- [ ] Secret exclusiva por webhook.
- [ ] Rotação de secret funcional.

---

## Retry e DLQ

- [ ] Retry segue progressão definida.
- [ ] Evento vai para DLQ após quinta falha.
- [ ] Replay administrativo funciona.

---

## Observabilidade

- [ ] Logs emitidos corretamente.
- [ ] Métricas expostas.
- [ ] Tracing disponível.

---

# 13. Riscos e Mitigação

| Risco | Impacto | Mitigação |
|---------|----------|------------|
| Crescimento da Outbox | Performance | Índices + retenção futura |
| Endpoint indisponível | Falhas de entrega | Retry + DLQ |
| Vazamento de secret | Segurança | Rotação de secret |
| Eventos duplicados | Reprocessamento indevido | X-Event-Id |
| Reinício da API | Interrupção da entrega | Worker separado |
| Payload excessivo | Consumo excessivo | Limite 64KB |
| Escala futura | Ordering comprometido | Estratégia futura de particionamento |

---