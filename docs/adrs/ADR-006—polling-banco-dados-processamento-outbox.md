# ADR — Polling Baseado em Banco de Dados para Processamento da Outbox

## Status

Aceita

---

## Contexto

Após a adoção do padrão Transactional Outbox e da utilização de um worker dedicado para processamento de eventos, a arquitetura precisava definir como o worker descobriria novos registros pendentes para entrega.

O requisito de produto define que notificações de mudança de status devem chegar ao cliente em menos de 10 segundos após a alteração do pedido.

Ao mesmo tempo, a equipe decidiu que a solução deve:

- Reaproveitar a infraestrutura existente.
- Evitar novos componentes operacionais.
- Manter simplicidade arquitetural.
- Preservar baixo custo de manutenção.

A questão arquitetural discutida foi:

> Como o worker será notificado sobre novos eventos registrados na Outbox?

As alternativas consideradas incluíram polling periódico da tabela de Outbox e mecanismos reativos baseados em recursos do banco de dados ou sistemas externos de mensageria.

---

## Decisão

O worker consumirá eventos através de **polling periódico da tabela de Outbox armazenada no MySQL**.

O processo executará continuamente um ciclo de leitura dos eventos pendentes.

Modelo conceitual:

```text
Loop do Worker
      ↓
Consultar Outbox
      ↓
Buscar eventos pendentes
      ↓
Processar eventos
      ↓
Atualizar status
      ↓
Aguardar próximo ciclo
```

A frequência definida para consulta será:

```text
2 segundos
```

O worker deverá:

1. Buscar eventos pendentes.
2. Priorizar eventos mais antigos.
3. Processar em lotes.
4. Atualizar o estado do evento após o processamento.

---

### Estruturas Impactadas

```text
webhook_outbox
```

A estrutura deverá permitir consultas eficientes através de índices adequados para:

```text
status
created_at
```

---

### Componentes Impactados

```text
src/worker.ts
src/modules/webhooks/webhook.processor.ts
src/modules/webhooks/webhook.repository.ts
```

---

### Fluxo Conceitual

```text
OrderService.changeStatus()
        ↓
webhook_outbox
        ↓
Worker (polling)
        ↓
Entrega HTTP
        ↓
Atualização do status do evento
```

---

## Alternativas Consideradas

### Alternativa 1 — Trigger de Banco para Notificar o Worker

#### Descrição

Utilizar triggers do banco para disparar processamento imediatamente após a gravação do evento.

Exemplo conceitual:

```text
INSERT na Outbox
        ↓
Trigger
        ↓
Acorda o Worker
```

#### Benefícios

- Menor latência.
- Comportamento mais próximo de processamento em tempo real.

#### Motivos para descarte

- O MySQL não possui mecanismo nativo equivalente ao modelo LISTEN/NOTIFY do PostgreSQL.
- Triggers executam SQL, mas não notificam processos externos.
- Exigiria mecanismos auxiliares artificiais para comunicação.
- Aumentaria a complexidade da solução.

#### Resultado

Alternativa descartada.

---

### Alternativa 2 — Redis Streams ou Sistema Externo de Mensageria

#### Descrição

Publicar os eventos em Redis Streams e utilizar consumidores dedicados.

Exemplo:

```text
Pedido
    ↓
Redis Streams
    ↓
Consumer
```

#### Benefícios

- Arquitetura mais orientada a eventos.
- Maior potencial de escalabilidade.
- Menor necessidade de polling.

#### Motivos para descarte

- Introduz nova infraestrutura.
- Exige operação adicional.
- Aumenta custo e complexidade.
- Considerado excessivo para o problema atual.

#### Resultado

Alternativa descartada.

---

### Alternativa 3 — Polling com Intervalos Maiores

#### Descrição

Executar consultas com intervalos mais longos, por exemplo:

```text
10s
30s
60s
```

#### Benefícios

- Menor carga sobre o banco.
- Menor atividade contínua do worker.

#### Motivos para descarte

- Latência incompatível com a expectativa dos clientes.
- Menor percepção de tempo real.
- Piora da experiência de integração.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Simplicidade Operacional

A solução utiliza apenas componentes já existentes:

```text
MySQL
Prisma
Node.js
Worker
```

Não há necessidade de:

```text
Redis
Kafka
RabbitMQ
Serviços adicionais
```

---

#### Atendimento ao SLA de Produto

Com polling a cada 2 segundos, a solução atende confortavelmente o requisito de entrega inferior a 10 segundos.

---

#### Menor Complexidade de Implementação

O mecanismo é simples de entender, operar e depurar.

O fluxo de processamento permanece completamente visível através do banco de dados.

---

#### Evolução Gradual

A estratégia pode evoluir futuramente sem alterar a semântica da Outbox.

Possíveis evoluções:

- Ajuste da frequência.
- Múltiplos workers.
- Transição futura para mensageria dedicada.

---

#### Maior Facilidade de Observabilidade

Toda a fila é persistida no banco.

Isso facilita:

- Consultas operacionais.
- Auditoria.
- Investigação de problemas.
- Métricas de backlog.

---

### Consequências Negativas

#### Latência Não Imediata

O processamento não ocorre no exato momento da criação do evento.

Existe um atraso inerente ao ciclo de polling.

Exemplo:

```text
Evento criado
 ↓
Aguarda próximo ciclo
 ↓
Evento processado
```

---

#### Consultas Contínuas ao Banco

Mesmo quando não houver eventos pendentes, o worker continuará executando consultas periódicas.

---

#### Dependência de Índices Adequados

Sem índices apropriados, a leitura contínua da Outbox pode degradar o desempenho ao longo do tempo.

---

#### Crescimento da Outbox

Quanto maior o volume de registros armazenados, maior a necessidade de manutenção da estrutura de dados e estratégias futuras de retenção.

---

## Trade-off Explícito

A decisão prioriza:

```text
Simplicidade
Baixo custo operacional
Reuso da infraestrutura existente
```

em vez de:

```text
Latência mínima possível
Arquitetura totalmente orientada a eventos
Escalabilidade imediata
```

A equipe considera aceitável abrir mão do processamento instantâneo em troca de uma solução significativamente mais simples de implementar e operar.

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões arquiteturais:

- Database Polling Pattern
- Transactional Outbox Consumer
- Background Processing
- Eventual Processing
- Batch Consumption