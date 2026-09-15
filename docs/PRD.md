# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature
Atualmente, clientes B2B precisam consultar periodicamente o endpoint de pedidos para identificar mudanças de status. Esse modelo de polling gera tráfego desnecessário, aumenta custos de integração dos clientes e cria latência na propagação das informações.

A feature de Sistema de Webhooks de Notificação de Pedidos permitirá que clientes cadastrados recebam notificações automáticas via HTTP sempre que ocorrerem mudanças de status em seus pedidos. O sistema será baseado em arquitetura Outbox Pattern, utilizando o banco MySQL existente, garantindo confiabilidade, desacoplamento da transação principal e entrega assíncrona dos eventos.

O objetivo é disponibilizar uma solução de notificação quase em tempo real, com latência inferior a 10 segundos, sem introduzir dependências adicionais de infraestrutura.

## Problema e motivação
Clientes B2B precisam frequentemente saber quando há mudança no status de seus pedidos. Atualmente, essas empresas dependem de consultas periódicas à API, gerando sobrecarga de infraestrutura, latência nas integrações e insatisfação. O atraso ou ausência de notificações pode culminar em perda de receita e migração para concorrentes.

## Público-alvo
- Clientes B2B integrados via API.
- Equipes técnicas dos clientes responsáveis pela integração.
- Administradores internos responsáveis por suporte e reprocessamento de falhas.

## Cenários de uso

- Cenário 1: Atualização logística:
Quando um pedido muda de PROCESSING para SHIPPED, o sistema do cliente recebe automaticamente uma notificação e inicia seus processos internos de logística.

- Cenário 2: Rastreamento de entregas:
O cliente configura o webhook para receber apenas eventos SHIPPED e DELIVERED, reduzindo volume de mensagens e focando em etapas críticas do fluxo.

- Cenário 3: Recuperação após indisponibilidade:
Caso o endpoint do cliente fique indisponível temporariamente, o sistema tenta reenviar automaticamente o evento utilizando política de retry com backoff exponencial.

- Cenário 4: Reprocessamento operacional:
Após uma falha permanente, administradores podem reprocessar eventos armazenados na DLQ por meio de endpoint administrativo.

## Objetivos
- Propiciar atualização de status "quase em tempo real" (<10s) para clientes B2B
- Eliminar a necessidade de polling contínuo pelos clientes.
- Reduzir tráfego desnecessário na API e custos de infraestrutura para todos os lados
- Aumentar confiança, satisfação e retenção das contas estratégicas
- Garantir entrega assíncrona resiliente a falhas temporárias.
- Elevar a segurança e integridade das integrações por meio de webhooks autenticados
- Fornecer mecanismos de auditoria e reprocessamento.

## Métricas de Sucesso
- Latência entre mudança de status e envio do webhook menor que 10 segundos
- Taxa de sucesso na primeira tentativa maior que 95%
- Nenhuma perda de eventos após commit da transação
- Disponibilidade do mecanismo de entrega maior igual a 99,9%
- Tempo de processamento do worker compatível com polling de 2 segundos
- Tempo máximo de timeout por envio de 10 segundos

## Escopo
- Gerenciamento de Webhooks
Criar webhook.
Atualizar webhook.
Excluir webhook.
Listar webhooks por cliente.
Ativar/desativar webhook.
Configurar filtros de status/eventos.

- Processamento de eventos
Geração de eventos de mudança de status.
Armazenamento na outbox.
Worker de processamento.
Entrega HTTP para endpoints externos.
Retry automático.
Dead Letter Queue (DLQ).
Reprocessamento manual.

- Segurança
HTTPS obrigatório.
Assinatura HMAC-SHA256.
Secret individual por endpoint.
Rotação de secret com período de convivência de 24h.

- Observabilidade
Histórico de entregas.
Registro de payload enviado.
Status de entrega.
Tempo de resposta.
Logs de auditoria do replay administrativo.

## Fora de Escopo
- Rate limiting de saída.
- Escalonamento para múltiplos workers.
- Garantia de ordering global.
- Infraestrutura baseada em Redis Streams, Kafka ou soluções externas.
- Estratégias exactly-once delivery.
- Envio de e-mails automáticos quando um webhook falhar ou for movido à DLQ (item postergado de propósito)
- Disponibilização de dashboard visual para acompanhamento dos webhooks pelos clientes (projeto do frontend separado)

## Requisitos Funcionais

### RF-01: Cadastro de Webhook

O sistema deve permitir o cadastro de webhooks para clientes.

#### Dados obrigatórios
- URL do endpoint (HTTPS obrigatório)
- customer_id
- Lista de status/eventos monitorados

#### Comportamento
- A secret deve ser gerada automaticamente pela plataforma.
- O webhook deve ser criado inicialmente como ativo.
- Cada webhook deve possuir uma secret exclusiva.

---

### RF-02: Atualização de Webhook

O sistema deve permitir a atualização das configurações de um webhook existente.

#### Campos editáveis
- URL
- Lista de eventos monitorados
- Estado do webhook (ativo/inativo, caso implementado)

---

### RF-03: Remoção de Webhook

O sistema deve permitir a remoção de um webhook através de endpoint dedicado.

#### Comportamento
- Após removido, o webhook não deve receber novos eventos.
- Eventos já entregues permanecem disponíveis no histórico.

---

### RF-04: Listagem de Webhooks

O sistema deve disponibilizar endpoint para consulta dos webhooks associados a um cliente.

#### Informações retornadas
- Identificador do webhook
- URL cadastrada
- Eventos monitorados
- Status do webhook
- Datas de criação e atualização

---

### RF-05: Geração de Eventos

Sempre que ocorrer uma mudança de status em um pedido, o sistema deve registrar um evento na Outbox.

#### Regras
- A criação do evento deve ocorrer na mesma transação que:
  - Atualiza a tabela de pedidos
  - Registra o histórico de status
  - Atualiza estoque
- Caso a gravação do evento falhe, toda a transação deve ser revertida.

---

### RF-06: Filtragem de Eventos

O sistema deve gerar eventos somente para os webhooks interessados naquele status específico.

#### Regra
- Se nenhum webhook do cliente estiver configurado para determinado status, nenhum evento deverá ser inserido na Outbox.

---

### RF-07: Snapshot do Payload

O payload do evento deve ser gerado e armazenado no momento da criação do evento.

#### Objetivo
Garantir que futuras alterações no pedido não modifiquem informações já registradas em eventos anteriores.

---

### RF-08: Processamento Assíncrono

O sistema deve processar eventos por meio de um worker dedicado.

### Regras
- O worker deve executar separadamente da API.
- O worker deve consultar a Outbox a cada 2 segundos.
- Apenas eventos pendentes devem ser processados.

---

## RF-09: Entrega de Eventos

O sistema deve enviar notificações HTTP para os endpoints cadastrados.

### Headers obrigatórios

```http
Content-Type: application/json
X-Event-Id
X-Signature
X-Timestamp
X-Webhook-Id
```

---

## RF-10: Retry Automático

O sistema deve reenviar eventos que falharem durante a entrega.

### Política de retry

| Tentativa | Espera |
|------------|---------|
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

Após a quinta falha, o evento deve ser movido para a DLQ.

---

## RF-11: Dead Letter Queue (DLQ)

O sistema deve armazenar eventos definitivamente falhos em uma tabela dedicada.

### Informações armazenadas
- Payload original
- Motivo da falha
- Data da falha
- Metadados de processamento

---

## RF-12: Replay de Eventos

O sistema deve permitir o reprocessamento manual de eventos presentes na DLQ.

### Endpoint

```http
POST /admin/webhooks/dead-letter/:id/replay
```

### Regras
- Somente usuários com role ADMIN podem executar a ação.
- O replay deve ser auditado.
- O evento deve ser reenfileirado como pendente.

---

## RF-13: Histórico de Entregas

O sistema deve disponibilizar consulta do histórico das tentativas de entrega.

### Endpoint

```http
GET /webhooks/:id/deliveries
```

### Informações exibidas
- Payload enviado
- Status da entrega
- Resposta recebida
- Tempo de resposta
- Timestamp da tentativa

---

## RF-14: Rotação de Secret

O sistema deve permitir a rotação da secret utilizada na assinatura dos webhooks.

### Regras
- A nova secret passa a ser válida imediatamente.
- A secret antiga permanece válida durante 24 horas.
- Após 24 horas a secret anterior deve ser invalidada automaticamente.

---

## RF-15: Estrutura do Payload

O sistema deve enviar payload JSON contendo:

```json
{
  "event_id": "uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-01-01T10:00:00Z",
  "order_id": "uuid",
  "order_number": "ORD-123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "uuid",
  "total_cents": 10000
}
```

### Regra
- Itens do pedido não devem ser enviados.
- Clientes devem utilizar a API de pedidos para obter detalhes adicionais.

---

# 7. Requisitos Não Funcionais

## RNF-01: Arquitetura

- Implementar Outbox Pattern utilizando MySQL.
- Não adicionar Redis, Kafka ou outra infraestrutura de mensageria nesta fase.

---

## RNF-02: Latência

- O sistema deve entregar eventos em menos de 10 segundos após a mudança de status.
- O polling do worker deve ocorrer a cada 2 segundos.

---

## RNF-03: Disponibilidade e Confiabilidade

- O sistema deve operar sob modelo de entrega At-Least-Once.
- Nenhum evento pode ser perdido após o commit da transação principal.

---

## RNF-04: Segurança

- Todo webhook deve utilizar HTTPS.
- Todo payload deve ser assinado utilizando HMAC-SHA256.
- Cada webhook deve possuir secret própria.

---

## RNF-05: Timeout

- Requisições HTTP realizadas pelo worker devem possuir timeout máximo de 10 segundos.

---

## RNF-06: Limite de Payload

- O tamanho máximo permitido para um payload é de 64 KB.
- Eventos acima desse limite devem falhar e ser tratados adequadamente.

---

## RNF-07: Observabilidade

- Todas as operações devem gerar logs utilizando Pino.
- Deve existir trilha de auditoria para replay de eventos.
- Histórico de entregas deve ser persistido.

---

## RNF-08: Compatibilidade Arquitetural

A implementação deve reutilizar componentes já existentes:

- Prisma
- AppError
- Middleware centralizado de erros
- Zod
- Estrutura modular da aplicação

---

# 8. Decisões e Trade-offs Principais

| Decisão | Justificativa | Trade-off |
|----------|-------------|-----------|
| Outbox Pattern em MySQL | Simplicidade operacional | Dependência de polling |
| Worker separado da API | Maior resiliência | Mais um processo para operar |
| Polling a cada 2 segundos | Atende requisito de latência | Existe atraso mínimo inevitável |
| Single Worker | Simplicidade e ordenação natural | Escalabilidade limitada |
| Entrega At-Least-Once | Alta confiabilidade | Possibilidade de eventos duplicados |
| Snapshot do payload | Preserva estado histórico | Maior uso de armazenamento |
| DLQ separada | Melhor suporte e auditoria | Mais estrutura de banco |
| Retry limitado a 5 tentativas | Evita eventos eternamente pendentes | Possibilidade de abandono após limite |
| HMAC-SHA256 | Padrão amplamente adotado | Necessidade de gestão de secrets |

---

# 9. Dependências

## Dependências Técnicas

- MySQL
- Prisma ORM
- Node.js
- Pino
- Zod
- JWT Authentication
- Middleware requireRole
- Infraestrutura atual da API de pedidos

---

## Dependências de Negócio

- Aprovação da equipe de segurança
- Disponibilização de documentação para clientes
- Endpoints consumidores operados pelos clientes

---

## Dependências Operacionais

- Deploy de processo worker separado
- Configuração de monitoramento
- Configuração de logs centralizados

---

# 10. Riscos e Mitigação

| Risco | Impacto | Mitigação |
|---------|----------|------------|
| Endpoint do cliente indisponível | Falha de entrega | Retry exponencial |
| Indisponibilidade prolongada do cliente | Eventos não entregues | DLQ + Replay |
| Vazamento de secret | Risco de fraude | Rotação com grace period |
| Crescimento excessivo da Outbox | Queda de performance | Índices e arquivamento futuro |
| Eventos duplicados | Processamento duplicado no cliente | X-Event-Id para deduplicação |
| Sobrecarga do banco | Lentidão de processamento | Leitura em batch e índices |
| Escalabilidade futura | Perda de ordering | Estratégias futuras de particionamento |
| Falha ao gravar na Outbox | Inconsistência de negócio | Mesma transação SQL |

---

# 11. Critérios de Aceitação

## Funcionais

- [ ] Cliente consegue criar webhooks via API.
- [ ] Cliente consegue listar webhooks.
- [ ] Cliente consegue atualizar webhooks.
- [ ] Cliente consegue remover webhooks.
- [ ] Filtros de eventos funcionam corretamente.
- [ ] Histórico de entregas é consultável.
- [ ] Replay de DLQ funciona para usuários ADMIN.

---

## Processamento

- [ ] Mudança de status gera evento na Outbox.
- [ ] Evento é criado dentro da mesma transação do pedido.
- [ ] Worker processa eventos pendentes.
- [ ] Eventos são entregues aos endpoints configurados.
- [ ] Retry segue exatamente a política definida.
- [ ] Evento é encaminhado para DLQ após cinco falhas.

---

## Segurança

- [ ] URLs HTTP são rejeitadas.
- [ ] HMAC-SHA256 é enviado corretamente.
- [ ] X-Event-Id é enviado em todas as notificações.
- [ ] X-Timestamp é enviado em todas as notificações.
- [ ] X-Webhook-Id é enviado em todas as notificações.
- [ ] Rotação de secret funciona conforme definido.

---

## Qualidade

- [ ] Payload respeita limite de 64 KB.
- [ ] Timeout máximo de envio é 10 segundos.
- [ ] Sistema opera em modelo At-Least-Once.
- [ ] Revisão de segurança foi aprovada.

---

# 12. Estratégia de Testes e Validação

## Testes Unitários

### Configuração de Webhooks
- Criação
- Atualização
- Remoção
- Validações de schema

### Segurança
- Geração de secret
- Rotação de secret
- Assinatura HMAC-SHA256
- Validação HTTPS

### Processamento
- Filtro de eventos
- Criação de payload snapshot
- Cálculo de backoff
- Controle de retries

---

## Testes de Integração

### Outbox
- Inserção durante mudança de status
- Operação dentro da transação
- Rollback em caso de falha

### Worker
- Leitura da Outbox
- Processamento em lote
- Atualização de status

### DLQ
- Movimentação após falhas
- Replay administrativo

---

## Testes End-to-End

### Fluxo Principal

```text
Mudança de status
→ Evento criado
→ Evento entra na Outbox
→ Worker processa
→ Webhook enviado
→ Entrega registrada
```

### Fluxos de Falha

```text
Falha no endpoint
→ Retry automático
→ Exaustão de tentativas
→ DLQ
→ Replay administrativo
```

---

## Testes de Carga

### Objetivos

- Validar crescimento da Outbox.
- Validar throughput do worker.
- Medir impacto do polling no MySQL.
- Simular grandes volumes de alterações de status.

---

## Testes de Segurança

### Verificações

- TLS obrigatório.
- Assinatura HMAC válida.
- Manipulação indevida de payload.
- Replay attack utilizando X-