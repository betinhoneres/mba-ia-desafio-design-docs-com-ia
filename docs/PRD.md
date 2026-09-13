# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo
O Sistema de Webhooks de Notificação de Pedidos do OMS permite que clientes B2B recebam notificações automáticas e seguras sempre que houver mudança relevante de status em pedidos. A solução elimina a necessidade de polling constante via API, reduz custos operacionais e melhora a experiência e retenção de clientes-chave.

## Problema
Clientes B2B precisam frequentemente saber quando há mudança no status de seus pedidos. Atualmente, essas empresas dependem de consultas periódicas à API, gerando sobrecarga de infraestrutura, latência nas integrações e insatisfação. O atraso ou ausência de notificações pode culminar em perda de receita e migração para concorrentes.

## Público-alvo
- Times de tecnologia (TI/engenharia) dos clientes B2B ou parceiros logísticos
- Parceiros que integram sistemas de ERP, logística, distribuição e automação

## Objetivos
- Propiciar atualização de status "quase em tempo real" (<10s) para clientes B2B
- Reduzir tráfego desnecessário na API e custos de infraestrutura para todos os lados
- Aumentar confiança, satisfação e retenção das contas estratégicas
- Elevar a segurança e integridade das integrações por meio de webhooks autenticados

## Métricas de Sucesso
- 95% dos eventos entregues em menos de 10 segundos de latência
- Redução de 70% nos acessos GET /orders de clientes aderentes
- Redução de churn por abandono dos parceiros B2B em pelo menos 50%
- Indice de falhas (permanentes) nos webhooks inferior a 1% a cada mês

## Requisitos Funcionais
1. Registro de eventos de status na outbox junto à transação do pedido
2. Worker externo à API com polling de 2 segundos processando a fila de notificações
3. Retry automático com backoff exponencial (1m, 5m, 30m, 2h, 12h), até 5 tentativas
4. Dead Letter Queue (DLQ) persistente para eventos não entregues, com reprocessamento manual (ADMIN)
5. Cadastro de endpoints de webhook por cliente, cada um com secret individual e rotacionável
6. Todos os eventos enviados são assinados via HMAC-SHA256; secret exposta via API (nunca no payload do evento)
7. Garantia de entrega at-least-once, usando event_id único por transmissão; documentação sobre deduplicação é obrigatória
8. Payload enxuto (sem itens do pedido), restrito a 64KB; eventos maiores são rejeitados
9. Somente URLs HTTPS aceitas para endpoints; validação feita nos schemas (Ex: Zod)
10. CRUD completo de webhooks com autenticação e autorização adequada (role ADMIN obrigatória para replay manual)

## Fora de Escopo
- Envio de e-mails automáticos quando um webhook falhar ou for movido à DLQ (item postergado de propósito)
- Disponibilização de dashboard visual para acompanhamento dos webhooks pelos clientes (projeto do frontend separado)

## Riscos
| Risco                                                      | Probabilidade | Impacto | Mitigação                                                                                  |
|-----------------------------------------------------------|:-------------:|:-------:|-------------------------------------------------------------------------------------------|
| Clientes sem tratamento de deduplicidade pelo event_id     |     Média     |  Alto   | Documentação exemplo, onboarding técnico proativo; comunicação direta com parceiros         |
| Crescimento inesperado de eventos causa lentidão/backs     |     Baixa     | Médio   | Uso de métricas em produção; escalabilidade horizontal do worker e implementação futura de rate limiting |

---

