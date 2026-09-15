# ADR — Uso de Dead Letter Queue para Falhas Permanentes

## Status

Aceita

---

## Contexto

O sistema de Webhooks de Notificação de Pedidos possui uma estratégia de entrega assíncrona baseada em Transactional Outbox e processamento por worker dedicado.

Como os eventos são enviados para sistemas externos operados pelos clientes, falhas são inevitáveis ao longo do ciclo de vida da solução, incluindo:

- Endpoint temporariamente indisponível.
- Manutenções planejadas do cliente.
- Configuração incorreta da URL.
- Problemas de DNS.
- Certificados inválidos.
- Timeouts de resposta.
- Falhas de rede.

A arquitetura já prevê tentativas automáticas de reenvio utilizando backoff exponencial, porém existe a necessidade de definir o comportamento do sistema quando todas as tentativas forem esgotadas.

A principal preocupação é evitar que eventos falhos permaneçam indefinidamente na Outbox, dificultando operações, monitoramento e suporte.

---

## Decisão

Será adotado o padrão **Dead Letter Queue (DLQ)** para armazenar eventos cuja entrega falhar permanentemente após o esgotamento da política de retry.

Após a última tentativa configurada:

```text
Evento
    ↓
Tentativas de entrega
    ↓
Falha permanente
    ↓
Dead Letter Queue
```

A DLQ será implementada como uma estrutura persistida separadamente da Outbox principal.

A equipe decidiu que os eventos não permanecerão indefinidamente na Outbox após o consumo de todas as tentativas disponíveis.

---

### Política de Retry

Antes de ser considerado definitivamente falho, cada evento seguirá a política de retry definida pela arquitetura:

| Tentativa | Intervalo |
|------------|------------|
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

Após a quinta falha consecutiva:

```text
status = FAILED
```

o evento será movido para a DLQ.

---

### Modelo Conceitual

```text
webhook_outbox
        ↓
worker
        ↓
retry
        ↓
sucesso
        ou
dead letter queue
```

---

### Dados Preservados

Cada registro da DLQ deverá manter informações suficientes para auditoria e reprocessamento.

Conceitualmente:

```text
event_id
payload
motivo da falha
timestamp
informações de entrega
```

A decisão da estrutura exata do schema pertence ao FDD.

---

### Reprocessamento

Eventos armazenados na DLQ poderão ser reenviados manualmente.

O sistema disponibilizará um endpoint administrativo para replay:

```http
POST /admin/webhooks/dead-letter/:id/replay
```

O replay deverá:

1. Registrar auditoria da ação.
2. Reenfileirar o evento para processamento.
3. Retornar o evento ao fluxo normal de entrega.

Somente usuários com role `ADMIN` poderão executar essa operação.

---

## Componentes Impactados

### Módulo

```text
src/modules/webhooks
```

### Componentes

```text
webhook.processor.ts
webhook.service.ts
webhook.repository.ts
worker.ts
```

### Estruturas Persistentes

```text
webhook_outbox
webhook_dead_letter
webhook_deliveries
```

---

## Alternativas Consideradas

### Alternativa 1 — Manter Eventos Falhos na Própria Outbox

#### Descrição

Após o esgotamento das tentativas, o evento permaneceria na Outbox principal com status de falha.

Exemplo:

```text
webhook_outbox

PENDING
DELIVERED
FAILED
```

#### Benefícios

- Menor quantidade de tabelas.
- Implementação mais simples.
- Menos estruturas para gerenciar.

#### Motivos para descarte

- Mistura eventos ativos e eventos definitivamente falhos.
- Dificulta consultas operacionais.
- Aumenta complexidade do worker.
- Dificulta processos de suporte e auditoria.
- Torna o reprocessamento menos explícito.

#### Resultado

Alternativa descartada.

---

### Alternativa 2 — Retry Infinito

#### Descrição

Continuar tentando entregar eventos indefinidamente utilizando backoff crescente.

Exemplo:

```text
Falha
 ↓
Retry
 ↓
Falha
 ↓
Retry
 ↓
...
```

#### Benefícios

- Reduz a necessidade de intervenção manual.
- Maior probabilidade de entrega eventual.

#### Motivos para descarte

- Eventos podem permanecer indefinidamente em processamento.
- Crescimento contínuo da fila.
- Dificuldade operacional para identificar problemas reais.
- Compromete previsibilidade do sistema.

#### Resultado

Alternativa descartada.

---

### Alternativa 3 — Descartar o Evento Após a Última Tentativa

#### Descrição

Após o esgotamento das tentativas, o evento seria removido permanentemente.

#### Benefícios

- Menor consumo de armazenamento.
- Fluxo operacional simples.

#### Motivos para descarte

- Perda definitiva de informação.
- Ausência de evidência para suporte.
- Impossibilidade de reprocessamento.
- Incompatível com requisitos de rastreabilidade.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Preservação de Evidências

Eventos que não puderam ser entregues permanecem disponíveis para investigação.

```text
Falha
 ↓
DLQ
 ↓
Análise
```

Isso simplifica atividades de suporte e troubleshooting.

---

#### Reprocessamento Controlado

A equipe operacional consegue recuperar eventos sem necessidade de alterar registros diretamente no banco.

---

#### Outbox Mais Simples

A Outbox permanece focada apenas em eventos ativos ou em processamento.

Isso reduz complexidade do fluxo principal.

---

#### Melhor Observabilidade

A quantidade de eventos na DLQ torna-se um indicador claro de problemas de integração.

Exemplo:

```text
DLQ crescendo
→ Problema de entrega
```

---

#### Compatibilidade com Padrões de Mercado

O uso de Dead Letter Queue é prática comum em arquiteturas orientadas a mensagens e processamento assíncrono.

---

### Consequências Negativas

#### Estrutura Persistente Adicional

A solução introduz uma nova tabela e novos fluxos operacionais.

---

#### Necessidade de Monitoramento

Não basta armazenar eventos na DLQ.

A equipe precisa monitorar:

- Volume de eventos falhos.
- Tempo de permanência na DLQ.
- Necessidade de replay.

---

#### Necessidade de Ferramentas Operacionais

Reprocessamento, auditoria e investigação passam a exigir funcionalidades administrativas adicionais.

---

#### Acúmulo de Dados

A DLQ crescerá ao longo do tempo.

Será necessário definir futuramente:

- Retenção.
- Limpeza.
- Arquivamento.

Essa definição foi explicitamente deixada fora do escopo da feature atual.

---

## Trade-off Explícito

A decisão prioriza:

```text
Rastreabilidade
Auditabilidade
Recuperação operacional
```

em detrimento de:

```text
Menor complexidade
Menor número de tabelas
Menor custo de armazenamento
```

Aceitamos o custo operacional de manter uma DLQ para garantir que falhas permanentes não resultem em perda de informação e possam ser tratadas posteriormente pela equipe.

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões arquiteturais:

- Dead Letter Queue Pattern
- Retry with Backoff
- Failure Isolation
- Operational Replay
- Recoverable Messaging