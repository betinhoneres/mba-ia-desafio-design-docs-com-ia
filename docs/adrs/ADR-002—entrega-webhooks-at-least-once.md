# ADR — Entrega de Webhooks com Semântica At-Least-Once

## Status

Aceita

---

## Contexto

O sistema de Webhooks de Notificação de Pedidos precisa garantir que eventos de mudança de status sejam entregues aos consumidores externos de forma confiável.

Como a entrega ocorre através de HTTP para sistemas de terceiros, existem diversos cenários de falha que estão fora do controle da plataforma:

- Endpoint do cliente indisponível.
- Timeout de rede.
- Erros temporários de infraestrutura.
- Reinício do worker durante o processamento.
- Falhas após o envio da requisição, mas antes da confirmação do processamento local.

Nesse contexto, o sistema precisa definir qual garantia de entrega será oferecida aos consumidores.

Durante a discussão arquitetural, a equipe avaliou que garantir entrega única (*exactly-once*) introduziria coordenação entre produtor e consumidor, aumentando significativamente a complexidade da solução. Por esse motivo, foi decidido adotar o modelo de confiabilidade mais amplamente utilizado em integrações baseadas em webhooks.

---

## Decisão

O sistema adotará semântica de entrega **At-Least-Once** para todos os webhooks.

Isso significa que:

- Todo evento persistido na Outbox será entregue uma ou mais vezes.
- O sistema priorizará não perder eventos.
- Duplicidade de entrega é considerada aceitável.
- Consumidores deverão ser capazes de processar eventos idempotentemente.

Para suportar essa estratégia, cada evento receberá um identificador único no momento em que for inserido na Outbox.

Esse identificador será enviado em todas as notificações através do header:

```http
X-Event-Id
```

O valor será um UUID único por evento.

Em caso de múltiplas entregas do mesmo evento, o consumidor poderá realizar deduplicação utilizando esse identificador. 

---

## Aplicação na Arquitetura

### Produção do Evento

No momento da inserção na Outbox:

```text
OrderService.changeStatus()
    ↓
publishWebhookEvent(...)
    ↓
webhook_outbox
    ↓
event_id (UUID)
```

O `event_id` passa a representar a identidade permanente daquele evento. 

---

### Entrega

Durante o processamento pelo worker:

```text
webhook_outbox
    ↓
webhook.processor
    ↓
HTTP POST
```

Os seguintes headers deverão acompanhar a requisição:

```http
X-Event-Id
X-Signature
X-Timestamp
X-Webhook-Id
```

---

### Retry

Caso a entrega falhe:

```text
Falha
 ↓
Retry automático
 ↓
Nova tentativa
```

Cada nova tentativa reutiliza o mesmo `event_id`.

Dessa forma, todas as tentativas representam o mesmo evento de negócio. 【1-4c3da3】

---

## Componentes Impactados

### Módulos

```text
src/modules/webhooks
src/modules/orders
```

### Componentes

```text
publishWebhookEvent(...)
webhook.processor.ts
worker.ts
```

### Estruturas Persistentes

```text
webhook_outbox
webhook_dead_letter
webhook_deliveries
```

O identificador do evento deverá ser persistido junto ao registro da Outbox para permitir rastreabilidade e deduplicação.

---

## Alternativas Consideradas

### Alternativa 1 — Entrega Exactly-Once

#### Descrição

Garantir que cada evento seja entregue exatamente uma única vez.

```text
Evento
 ↓
Uma única entrega garantida
 ↓
Nunca duplicado
```

#### Benefícios

- Elimina necessidade de deduplicação pelo consumidor.
- Modelo aparentemente mais simples para os clientes.

#### Motivos para descarte

- Não existe suporte nativo em HTTP para exactly-once.
- Exigiria coordenação entre produtor e consumidor.
- Aumenta complexidade operacional.
- Amplia necessidade de controle de estado distribuído.
- Eleva significativamente o esforço de implementação.

A equipe avaliou que o ganho não justificaria a complexidade adicional para esta feature.

#### Resultado

Alternativa descartada.

---

### Alternativa 2 — Entrega Best-Effort (Sem Retry)

#### Descrição

Tentar entregar apenas uma vez e descartar o evento em caso de falha.

```text
Evento
 ↓
Tentativa única
 ↓
Sucesso ou perda definitiva
```

#### Benefícios

- Implementação extremamente simples.
- Menor necessidade de armazenamento.

#### Motivos para descarte

- Perda definitiva de eventos.
- Baixa confiabilidade.
- Incompatível com requisitos de negócio.
- Não atende clientes que dependem das notificações para automação operacional.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Prioridade para confiabilidade

A arquitetura privilegia a não perda de eventos.

```text
Evento criado
→ Evento eventualmente entregue
```

Mesmo diante de falhas temporárias, o sistema continuará tentando a entrega. 

---

#### Compatibilidade com padrões de mercado

A estratégia adotada segue o mesmo modelo utilizado por diversas plataformas de integração baseadas em webhooks.

A abordagem é amplamente conhecida por equipes de integração e arquitetos de software. 

---

#### Simplicidade arquitetural

Evita mecanismos complexos de coordenação distribuída.

A solução permanece compatível com:

```text
MySQL
Outbox Pattern
Worker Assíncrono
DLQ
```

---

#### Reprocessamento seguro

Eventos podem ser reenviados para processamento sem necessidade de mecanismos adicionais de sincronização.

A mesma estratégia vale para:

```text
Retry automático
Replay manual
```

---

### Consequências Negativas

#### Possibilidade de duplicidade

O mesmo evento pode ser recebido mais de uma vez.

Exemplo:

```text
Tentativa #1
↓
Cliente processa

Timeout na confirmação

↓

Tentativa #2
↓
Cliente recebe novamente
```

---

#### Responsabilidade compartilhada com o consumidor

Os consumidores precisam implementar deduplicação utilizando:

```http
X-Event-Id
```

A plataforma não garante processamento único do lado do cliente. 

---

#### Complexidade de documentação

A semântica de entrega precisa estar claramente documentada para evitar interpretações incorretas sobre o comportamento da integração.

Clientes devem ser orientados a tratar webhooks como eventos potencialmente duplicáveis.

---

## Trade-off Explícito

A decisão prioriza:

```text
Confiabilidade
```

em vez de:

```text
Unicidade de entrega
```

Aceitamos a possibilidade de eventos duplicados para reduzir o risco de perda de notificações.

O racional da decisão pode ser resumido da seguinte forma:

```text
Perder um evento é pior que entregar duas vezes.
```

Por esse motivo, a plataforma garante **At-Least-Once Delivery** e delega a deduplicação ao consumidor através do `X-Event-Id`. 

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões:

- At-Least-Once Delivery
- Idempotent Consumer
- Retry with Backoff
- Event Identifier Pattern
- Dead Letter Queue

---