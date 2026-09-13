# ADR 002: Política de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks

## Status
Decidido

## Contexto
O envio de webhooks pode falhar por motivos externos (instabilidade do endpoint do cliente, timeouts, falhas de rede). É necessário garantir tentativas suficientes de reenvio sem bloquear indefinidamente recursos do sistema. Também é preciso rastrear falhas permanentes e permitir reprocessamento.

## Decisão
Implementar política de até 5 tentativas de envio com backoff exponencial de acordo com a sequência 1m, 5m, 30m, 2h, 12h. Se todas as tentativas falharem, o evento é movido para uma Dead Letter Queue (DLQ), implementada como tabela separada `webhook_dead_letter`. Replays só podem ser feitos por usuário com role ADMIN via endpoint protegido.

## Alternativas Consideradas
- Retry indefinido (riscos de fila travada caso endpoint do cliente desapareça).
- Fewer retries (3), considerado insuficiente para indisponibilidades temporárias.
- Truncar eventos falhos sem rastreamento em DLQ.

## Consequências
- Protege o sistema de eventos engargalados.
- Permite auditoria e recuperação manual em casos críticos.
- Aumenta robustez e transparência da integração.
- O worker que realiza envios/controle estará em arquivo separado (ver ADR 005).
- Endpoints de replay estarão em módulo webhooks (ex: `src/modules/webhooks`).
