# ADR 001: Uso do Padrão Outbox no MySQL para Envio de Webhooks

## Status
Decidido

## Contexto
Clientes B2B demandaram notificações em tempo quase real sobre mudanças de status de pedidos. O sistema atual exige consultas REST frequentes ao endpoint `GET /orders`, o que é ineficaz e custoso. Enviar webhooks diretamente da transação de update no pedido é arriscado (clientes lentos, indisponíveis ou falhas de rede podem afetar transação principal).

## Decisão
Empregar o padrão Outbox no banco de dados MySQL já existente. Quando um status de pedido muda, a aplicação insere um evento em uma tabela `webhook_outbox` dentro da mesma transação que atualiza o pedido. Um worker depois processa e envia os eventos de forma assíncrona.

## Alternativas Consideradas
- Envio síncrono de webhooks dentro do transaction handler do `order.service.ts` (riscos de latência alta e inconsistente, rollback indesejado).
- Implementar Redis Streams ou brokers externos.
- Escalar bancos para receber triggers ou utilizar recursos do Postgres como NOTIFY/LISTEN (infraestrutura não disponível).

## Consequências
- Evita inconsistências entre update de pedidos e envio dos eventos.
- Simplifica infra: usando apenas o MySQL já existente.
- Facilita retries e DLQ.
- Paths relevantes: lógica principal de mudança de status fica em `src/modules/orders/order.service.ts`; integração com Outbox será adicionada a esse arquivo. Modelos Prisma em `prisma/schema.prisma`.
