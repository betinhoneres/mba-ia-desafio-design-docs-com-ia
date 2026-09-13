# ADR 004: Garantia de entrega At-least-once com Uso de X-Event-Id para Idempotência

## Status
Decidido

## Contexto
Falhas de comunicação, timeouts ou retries podem causar múltiplos envios do mesmo evento de webhook ao cliente. Uma garantia de exactly-once exigiria coordenação extra e complexa, além de depender do receptor. Sistemas como Stripe, GitHub, Shopify seguem o padrão de at-least-once, transferindo deduplicação para o cliente.

## Decisão
O sistema garante entrega at-least-once de eventos de webhook. Cada evento terá um `event_id` único, enviado no header `X-Event-Id`. Os clientes devem implementar deduplicação a partir desse identificador, usando-o para ignorar replays e garantir idempotência do lado receptor.

## Alternativas Consideradas
- Tried global ordering or exactly-once (exigiria coordenação entre as partes, counters, confirmações; complexidade e custo grande, além de não ser expectativa dos clientes).
- Nenhum controle (risco alto de efeitos colaterais, clientes processando múltiplas vezes eventos idênticos).

## Consequências
- Simplicidade e alinhamento com práticas amplamente aceitas no mercado.
- A documentação oficial (portal do desenvolvedor) deve alertar e exemplificar deduplicação por event_id.
- O event_id é gerado e persistido junto ao evento (ex: `src/modules/webhooks` e modelo de webhook_outbox em `prisma/schema.prisma`).
