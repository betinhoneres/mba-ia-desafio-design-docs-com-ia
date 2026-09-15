# ADR — Adoção de Transactional Outbox para Eventos de Pedidos

## Status

Aceita

---

## Contexto

A plataforma necessita implementar um mecanismo de notificações em tempo real para clientes B2B quando houver mudança de status em pedidos.

O fluxo atual de alteração de status já executa uma transação composta por múltiplas operações críticas:

- Atualização da tabela de pedidos (`orders`)
- Registro do histórico de status (`order_status_history`)
- Atualização de estoque

A nova funcionalidade de webhooks introduz a necessidade de publicar eventos sempre que ocorrer uma mudança de status.

O principal requisito arquitetural é garantir que:

- Nenhum evento seja perdido após uma alteração de status concluída com sucesso.
- Nenhuma alteração de status seja confirmada sem que o evento correspondente tenha sido registrado.
- A disponibilidade ou lentidão do endpoint do cliente não afete o fluxo principal de negócio.

Durante a discussão técnica foi explicitamente descartada a execução síncrona do webhook dentro da transação de negócio devido ao risco de bloquear operações críticas e introduzir dependências externas no processo de atualização do pedido.

Além disso, a equipe definiu que a solução deve reutilizar a infraestrutura existente baseada em MySQL e Prisma, evitando a introdução de componentes adicionais de mensageria.

---

## Decisão

Adotar o padrão **Transactional Outbox** para a publicação de eventos de mudança de status de pedidos.

A solução consiste em registrar um evento em uma tabela de Outbox dentro da mesma transação responsável pela atualização do pedido.

Conceitualmente:

```text
changeStatus()
 ├─ update orders
 ├─ insert order_status_history
 ├─ update stock
 └─ insert webhook_outbox
```

Todos os passos participam da mesma transação do banco.

Após o commit da transação:

1. O evento permanece registrado na Outbox.
2. Um worker assíncrono independente consulta a Outbox.
3. O worker realiza a entrega dos webhooks.
4. O estado do evento é atualizado conforme sucesso ou falha.

A integração ocorrerá no fluxo de mudança de status do pedido através de uma função dedicada de publicação de eventos, executada dentro da transação corrente.

Referências explícitas mencionadas durante a reunião:

### Módulos impactados

```text
src/modules/orders
src/modules/webhooks
```

### Componentes impactados

```text
OrderService.changeStatus()
publishWebhookEvent(...)
```

### Estruturas persistentes

```text
orders
order_status_history
webhook_outbox
```

O evento armazenado na Outbox deverá representar um snapshot do momento da mudança de status, evitando dependência de futuras modificações do pedido. 

---

## Alternativas Consideradas

### Alternativa 1 — Chamada Síncrona para o Endpoint do Cliente

#### Descrição

Executar a requisição HTTP para o webhook diretamente durante a execução do fluxo de alteração de status.

```text
changeStatus()
 ├─ update order
 ├─ update history
 ├─ update stock
 └─ HTTP POST webhook
```

#### Motivos para descarte

- Acoplamento da operação de negócio à disponibilidade do consumidor.
- Aumento da duração da transação.
- Possibilidade de degradação do sistema por clientes lentos.
- Dificuldade para lidar com falhas sem comprometer a transação principal.
- Necessidade de decidir entre rollback ou inconsistência caso o webhook falhe.

#### Resultado

Alternativa descartada. 

---

### Alternativa 2 — Redis Streams ou Sistema Externo de Mensageria

#### Descrição

Publicar os eventos em um mecanismo dedicado de mensageria, como Redis Streams.

#### Benefícios

- Maior escalabilidade.
- Arquitetura orientada a eventos mais robusta para crescimento futuro.

#### Motivos para descarte

- Introduz nova infraestrutura operacional.
- Aumenta custo de manutenção.
- Adiciona complexidade incompatível com o escopo atual.
- O benefício imediato não justifica o esforço adicional.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Consistência transacional

Se a alteração do pedido for persistida, o evento correspondente também será persistido.

```text
Pedido confirmado
→ Evento garantidamente registrado
```

Não existe cenário em que o pedido seja atualizado com sucesso e o evento não seja registrado.

#### Desacoplamento do fluxo de negócio

A mudança de status deixa de depender da disponibilidade dos consumidores externos.

Problemas nos endpoints dos clientes não interrompem operações críticas do sistema. 

#### Simplicidade operacional

A solução reutiliza:

```text
MySQL existente
Prisma existente
Infraestrutura atual
```

sem necessidade de Redis, Kafka ou outro broker. 

#### Evolução incremental

Permite adicionar futuramente:

- múltiplos workers;
- particionamento;
- novas estratégias de entrega;

sem alterar a semântica da geração do evento.

---

### Consequências Negativas

#### Dependência de polling

A entrega deixa de ser imediata.

O processamento passa a depender da frequência de leitura da Outbox pelo worker. 

#### Crescimento da tabela de Outbox

A tabela pode aumentar significativamente ao longo do tempo.

Será necessário definir futuramente políticas de retenção e arquivamento.

#### Complexidade maior que chamada direta

Apesar de mais confiável, a solução introduz:

- tabela adicional;
- worker dedicado;
- estados de processamento;
- reprocessamento de falhas.

#### Consistência eventual para entrega

O commit da transação e a entrega do webhook deixam de ocorrer no mesmo instante.

Existe um intervalo esperado entre:

```text
Status alterado
→ Evento persistido
→ Evento entregue
```

O trade-off é aceito pela equipe porque o requisito do produto considera aceitável uma latência inferior a 10 segundos. 【1-28d728】

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões arquiteturais:

- Transactional Outbox Pattern
- Asynchronous Processing
- Event-Driven Integration
- Event Snapshot Persistence

---