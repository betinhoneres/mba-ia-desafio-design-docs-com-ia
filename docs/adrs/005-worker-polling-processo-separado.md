# ADR 005: Worker de Webhooks em Processo Separado com Polling

## Status
Decidido

## Contexto
A entrega de webhooks não pode ser síncrona nem depender do processo principal da API: delays ou pausas nesse fluxo podem travar o sistema para todos os usuários. Uma execução desacoplada permite maior controle de concorrência, paralelismo nas tentativas e independência operacional. Polling é suficiente para atender o SLA de "menos de 10 segundos de latência". 

## Decisão
Um worker separado, executado via entry-point próprio (`src/worker.ts`), será responsável por ler a tabela outbox e processar os envios periodicamente a cada 2 segundos. Esse worker usa o mesmo banco e Prisma, mas é um processo Node distinto da aplicação principal (`src/server.ts`). A lógica de polling e processamento pode residir em `src/modules/webhooks/webhook.worker.ts` (ou similar).

## Alternativas Consideradas
- Priorizar triggers de banco (MySQL não oferece mecanismo como Postgres NOTIFY/LISTEN, e triggers só executam SQL).
- Usar filas externas (Redis Streams, SQS, etc — considerado overengineering no contexto, dado o escopo e tamanho da equipe).
- Incluir worker no mesmo processo da API (risco de interrupções e menos resiliência).

## Consequências
- Permite paralelismo e isolação de falhas.
- Facilmente escalável no futuro (novos workers = novos processos).
- Atende os requisitos de latência do negócio.
- Atenção: o worker requer uso da mesma configuração de banco e instância independente de Prisma (`src/config/database.ts`).
