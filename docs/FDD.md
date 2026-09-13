# FDD — Webhooks de Notificação de Pedidos no OMS (Order Management System)

## Sumário
- Fluxos principais do sistema
- Contratos públicos (endpoints)
- Matriz de erros WEBHOOK_
- Integração com o sistema existente
- Estratégias de resiliência e observabilidade

---

## 1. Fluxos do Sistema

### 1.1 Fluxo Outbox
- Sempre que um pedido tem seu status alterado (`src/modules/orders/order.service.ts`), um evento é inserido na tabela `webhook_outbox` dentro da mesma transação.
- O payload do evento é armazenado já "renderizado" com snapshot dos dados do pedido e metadados (event_id, data/hora, etc).

### 1.2 Fluxo Worker
- Processo separado em `src/worker.ts`, com lógica em `src/modules/webhooks/webhook.worker.ts`.
- Faz polling a cada 2 segundos, carrega eventos pendentes da outbox e executa o disparo HTTP.
- Atualiza status do evento para "enviado", "falhou", "processando" conforme resultado da entrega.

### 1.3 Fluxo Retry
- Em caso de falha, reagenda tentativa após período crescente (1m, 5m, 30m, 2h, 12h). Máximo de 5 tentativas.

### 1.4 Fluxo DLQ
- Após 5 falhas, evento vai para tabela `webhook_dead_letter`.
- Pode ser reprocessado manualmente via endpoint de replay ADMIN.
- Histórico e tentativas ficam rastreados para auditoria.

---

## 2. Contratos Públicos

### 2.1 Cadastro de Webhook
`POST /webhooks`

**Request:**
```json
{
  "url": "https://erp-cliente.com/oms-events",
  "status_list": ["PAID", "SHIPPED"]
}
```
Headers: `Authorization: Bearer <jwt>`

**Response 201:**
```json
{
  "id": "wh_82faf7d5",
  "url": "https://erp-cliente.com/oms-events",
  "status_list": ["PAID", "SHIPPED"],
  "secret": "****Gerada pela API****"
}
```

### 2.2 Consulta de Histórico de Entregas
`GET /webhooks/{webhook_id}/deliveries?limit=100`

**Response 200:**
```json
[
  {
    "event_id": "evt_86e491d3",
    "delivery_id": "del_9bfb1bf8",
    "sent_at": "2024-06-10T09:15:12Z",
    "status": "SUCCESS",
    "response_code": 200,
    "latency_ms": 612
  },
  {
    "event_id": "evt_41e867ca",
    "delivery_id": "del_075ad02c",
    "sent_at": "2024-06-10T08:42:11Z",
    "status": "FAILED",
    "response_code": 0,
    "latency_ms": null
  }
]
```

### 2.3 Replay Manual de Evento na DLQ
`POST /admin/webhooks/dead-letter/{event_id}/replay`
Headers: `Authorization: Bearer <jwt com role ADMIN>`

**Response 200:**
```json
{
  "event_id": "evt_2c2d1f33",
  "status": "REQUEUED"
}
```

### 2.4 Rotação de Secret do Webhook
`POST /webhooks/{webhook_id}/rotate-secret`

**Response 200:**
```json
{
  "webhook_id": "wh_82faf7d5",
  "old_secret_valid_until": "2024-06-11T10:00:00Z",
  "secret": "****Nova****"
}
```

---

## 3. Matriz de Erros (WEBHOOK_*)
| Código de Erro              | Situação                                                                                                                                     |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| WEBHOOK_INVALID_URL        | URL do webhook enviada não é HTTPS ou inválida                                                                                                |
| WEBHOOK_SECRET_REQUIRED    | Tentativa de cadastro/rotação sem informar secret (ou corrompida na base)                                                                      |
| WEBHOOK_INVALID_SIGNATURE  | Assinatura HMAC não valida para o payload entregue                                                                                            |
| WEBHOOK_NOT_FOUND          | Webhook não existe ou não pertence ao cliente                                                                                                 |
| WEBHOOK_DEAD_LETTER        | Evento movido para DLQ após tentativas excedidas                                                                                              |
| WEBHOOK_LIMIT_REACHED      | Limite de webhooks ativos por cliente excedido                                                                                               |
| WEBHOOK_PAYLOAD_TOO_LARGE  | Evento gerou payload superior a 64KB                                                                                                          |
| WEBHOOK_DELIVERY_TIMEOUT   | Timeout de 10 segundos ao entregar event                                                                                                      |
| WEBHOOK_REPLAY_UNAUTHORIZED| Usuário sem permissão ADMIN tentou replay                                                                                                     |
| WEBHOOK_DISABLED           | Webhook desativado ou marcado como inativo                                                                                                    |

---

## 4. Integração com o Sistema Existente
- **src/modules/orders/order.service.ts:**
  - Função de mudança de status será estendida para inserir registros na outbox (`webhook_outbox`) na mesma transação das alterações das ordens.
- **src/modules/webhooks/webhook.worker.ts:**
  - Nova lógica para polling, envio, retry/backoff e gestão de status de eventos (commit FINAL das entregas).
- **src/server.ts / src/worker.ts:**
  - Novo entry-point do worker separado da main API, ambos conectando ao mesmo banco via Prisma Client (config em `src/config/database.ts`).
- **prisma/schema.prisma:**
  - Modelagem ampliada com tabelas webhook_outbox, webhook_dead_letter, webhook_config (por endpoint), incluindo campos para status/tentativas/last_error.

---

## 5. Estratégias de Resiliência e Observabilidade
- **Retry + DLQ:** Garantia at-least-once, buffer para instabilidade e manual intervention via replay.
- **Timeout fixo**: 10 segundos por entrega; falhas tratadas como retry.
- **Logs estruturados:** Utilizar Pino (`src/shared/logger/index.ts`) para todas as operações críticas (in/outbound, entrega, retries, erros, replay, DLQ).
- **Tracing:** Incluir event_id e delivery_id em todos os logs e rastrear headers (X-Event-Id, X-Webhook-Id) nas requisições outbound.
- **Métricas:** Contagem de entregas, latência, retries, fails e DLQ expostos via endpoint privado ou serviço de monitoramento dedicado.
- **Auditoria:** Marcações de usuário quando houver replay manual via endpoint ADMIN, incluindo timestamp, requester, e payload reprocessado.

---

**Fim do FDD**

