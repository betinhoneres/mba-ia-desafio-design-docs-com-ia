# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

## Objetivo

Este documento mapeia requisitos, decisões arquiteturais, contratos, restrições, critérios de aceite e integrações documentadas nos artefatos da feature para suas respectivas origens.

O objetivo é garantir rastreabilidade completa entre:

- PRD
- RFC
- ADRs
- FDD
- Transcrição da reunião
- Código existente mencionado durante a reunião

---

## Legenda

### Fontes possíveis

| Valor | Significado |
|---------|---------|
| TRANSCRICAO | Informação extraída diretamente da reunião |
| CODIGO | Referência a arquivos, módulos ou componentes do sistema mencionados durante a reunião |

---

# Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|------|------|------|------|------|------|
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes desejam notificações em tempo real para mudanças de status | TRANSCRICAO | [09:00] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência aceitável inferior a 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-01 | docs/PRD.md | Restrição | Escopo limitado a webhooks outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-02 | docs/PRD.md | Restrição | Dashboard visual fora do escopo | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-03 | docs/PRD.md | Restrição | Notificações por email fora do escopo da fase atual | TRANSCRICAO | [09:37] Larissa |
| RFC-DEC-01 | docs/RFC.md | Decisão | Uso do padrão Transactional Outbox | TRANSCRICAO | [09:06] Diego |
| RFC-DEC-02 | docs/RFC.md | Decisão | Reutilização do MySQL existente | TRANSCRICAO | [09:07] Diego |
| RFC-DEC-03 | docs/RFC.md | Decisão | Worker dedicado para processamento | TRANSCRICAO | [09:11] Diego |
| RFC-DEC-04 | docs/RFC.md | Decisão | Polling periódico da Outbox | TRANSCRICAO | [09:09] Diego |
| RFC-DEC-05 | docs/RFC.md | Decisão | Polling executado a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| RFC-DEC-06 | docs/RFC.md | Decisão | Garantia At-Least-Once | TRANSCRICAO | [09:24] Diego |
| RFC-DEC-07 | docs/RFC.md | Decisão | HMAC-SHA256 para assinatura dos webhooks | TRANSCRICAO | [09:20] Sofia |
| RFC-DEC-08 | docs/RFC.md | Decisão | Uso de DLQ para falhas permanentes | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Chamada síncrona durante changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams como infraestrutura de mensageria | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco como mecanismo de notificação | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Exactly-Once Delivery | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Estratégia futura para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Rate limiting de saída | TRANSCRICAO | [09:38] Diego |
| ADR-001 | docs/adrs/ADR-Transactional-Outbox.md | Decisão | Transactional Outbox para eventos de pedidos | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-At-Least-Once.md | Decisão | Semântica At-Least-Once | TRANSCRICAO | [09:24] Diego |
| ADR-003 | docs/adrs/ADR-Worker-Dedicado.md | Decisão | Worker executado em processo independente | TRANSCRICAO | [09:11] Diego |
| ADR-004 | docs/adrs/ADR-HMAC-SHA256.md | Decisão | Assinatura HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-DLQ.md | Decisão | Dead Letter Queue para falhas permanentes | TRANSCRICAO | [09:18] Diego |
| ADR-006 | docs/adrs/ADR-Polling.md | Decisão | Polling baseado em banco de dados | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Evento criado dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Rollback caso inserção da Outbox falhe | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker consome eventos pendentes | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Eventos falhos seguem política de retry | TRANSCRICAO | [09:15] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Eventos vão para DLQ após esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Replay de eventos via endpoint administrativo | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato Público | POST /webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato Público | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato Público | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato Público | GET /webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato Público | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato Público | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato Público | Header X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato Público | Header X-Signature | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato Público | Header X-Timestamp | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato Público | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato Público | Payload order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Restrição | Não enviar itens do pedido no webhook | TRANSCRICAO | [09:43] Diego |
| FDD-SEC-01 | docs/FDD.md | Requisito Não Funcional | HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| FDD-SEC-02 | docs/FDD.md | Requisito Não Funcional | Secret exclusiva por webhook | TRANSCRICAO | [09:21] Sofia |
| FDD-SEC-03 | docs/FDD.md | Requisito Não Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-RS-01 | docs/FDD.md | Resiliência | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-RS-02 | docs/FDD.md | Resiliência | Retry exponencial | TRANSCRICAO | [09:15] Diego |
| FDD-RS-03 | docs/FDD.md | Resiliência | 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| FDD-RS-04 | docs/FDD.md | Resiliência | DLQ após quinta falha | TRANSCRICAO | [09:17] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Histórico de entregas para clientes | TRANSCRICAO | [09:34] Marcos |
| FDD-ERR-01 | docs/FDD.md | Matriz de Erros | Prefixo WEBHOOK_* | TRANSCRICAO | [09:28] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão do fluxo changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Processo dedicado do worker | CODIGO | src/worker.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Entry point principal da API | CODIGO | src/server.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Módulo de webhooks | CODIGO | src/modules/webhooks |
| FDD-INT-05 | docs/FDD.md | Integração | Processamento de eventos de webhook | CODIGO | src/modules/webhooks/webhook.processor.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Alternativa de implementação do worker citada na reunião | CODIGO | src/modules/webhooks/webhook.worker.ts |
| FDD-INT-07 | docs/FDD.md | Compatibilidade | Reutilização de AppError | CODIGO | AppError |
| FDD-INT-08 | docs/FDD.md | Compatibilidade | Reutilização de PrismaClient | CODIGO | PrismaClient |
| FDD-INT-09 | docs/FDD.md | Compatibilidade | Reutilização do logger Pino | CODIGO | Pino |
| FDD-INT-10 | docs/FDD.md | Compatibilidade | Reutilização de requireRole para ADMIN | CODIGO | requireRole |
| AC-01 | docs/FDD.md | Critério de Aceite | Evento criado na mesma transação do pedido | TRANSCRICAO | [09:40] Bruno |
| AC-02 | docs/FDD.md | Critério de Aceite | Polling executado a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| AC-03 | docs/FDD.md | Critério de Aceite | Retry segue política definida | TRANSCRICAO | [09:17] Diego |
| AC-04 | docs/FDD.md | Critério de Aceite | Evento enviado para DLQ após esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| AC-05 | docs/FDD.md | Critério de Aceite | Assinatura HMAC-SHA256 obrigatória | TRANSCRICAO | [09:20] Sofia |
| AC-06 | docs/FDD.md | Critério de Aceite | HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| AC-07 | docs/FDD.md | Critério de Aceite | Replay restrito a ADMIN | TRANSCRICAO | [09:36] Sofia |
| AC-08 | docs/FDD.md | Critério de Aceite | Worker separado da API principal | TRANSCRICAO | [09:11] Diego |
| AC-09 | docs/FDD.md | Critério de Aceite | Garantia At-Least-Once | TRANSCRICAO | [09:24] Diego |
| AC-10 | docs/FDD.md | Critério de Aceite | X-Event-Id utilizado para deduplicação | TRANSCRICAO | [09:25] Diego |

---

# Cobertura

## Documentos Cobertos

- docs/PRD.md
- docs/RFC.md
- docs/FDD.md
- docs/adrs/ADR-Transactional-Outbox.md
- docs/adrs/ADR-At-Least-Once.md
- docs/adrs/ADR-Worker-Dedicado.md
- docs/adrs/ADR-HMAC-SHA256.md
- docs/adrs/ADR-DLQ.md
- docs/adrs/ADR-Polling.md

---