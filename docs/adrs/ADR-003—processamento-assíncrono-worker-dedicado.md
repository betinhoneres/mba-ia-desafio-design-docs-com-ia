# ADR — Processamento Assíncrono via Worker Dedicado

## Status

Aceita

---

## Contexto

A arquitetura de Webhooks de Notificação de Pedidos necessita processar eventos registrados na Outbox e realizar entregas HTTP para sistemas de terceiros.

Durante a definição da solução, foi identificado que o processamento dos webhooks não deve compartilhar o mesmo ciclo de vida da API principal.

A preocupação principal é garantir que:

- Reinicializações da API não interrompam o processamento de eventos.
- O processamento das notificações permaneça desacoplado das requisições HTTP recebidas pela aplicação.
- A entrega de webhooks não impacte o desempenho dos endpoints de negócio.
- O sistema seja capaz de continuar evoluindo sem acoplar responsabilidades distintas em um único processo.

Além disso, a equipe já definiu a adoção do padrão Transactional Outbox para persistência dos eventos. Como consequência, torna-se necessário um mecanismo responsável por consumir a Outbox e executar as entregas. Esse papel será desempenhado por um worker dedicado.

---

## Decisão

A plataforma utilizará um **worker assíncrono executado como processo independente da API principal**.

Arquiteturalmente, teremos dois processos distintos:

```text
+------------------+
| API HTTP         |
| src/server.ts    |
+------------------+
          |
          v
+------------------+
| MySQL            |
+------------------+
          ^
          |
+------------------+
| Worker           |
| src/worker.ts    |
+------------------+
```

O worker será responsável por:

- Consultar eventos pendentes na Outbox.
- Executar entregas HTTP dos webhooks.
- Registrar o resultado das entregas.
- Executar políticas de retry.
- Encaminhar eventos para DLQ quando necessário.

A API continuará responsável apenas por:

- Receber requisições.
- Processar regras de negócio.
- Registrar eventos na Outbox.

A lógica de processamento dos webhooks permanecerá dentro do módulo de webhooks, enquanto o worker atuará apenas como ponto de entrada do processo.

### Estrutura esperada

```text
src/
├── server.ts
├── worker.ts
└── modules/
    └── webhooks/
        ├── webhook.processor.ts
        ├── webhook.service.ts
        ├── webhook.repository.ts
        ├── webhook.routes.ts
        └── webhook.schemas.ts
```

### Infraestrutura compartilhada

O worker reutilizará:

```text
Prisma
DATABASE_URL
Pino
AppError
Configurações da aplicação
```

Entretanto, o worker possuirá sua própria instância do `PrismaClient`, uma vez que será executado em um processo independente.

---

## Alternativas Consideradas

### Alternativa 1 — Executar o Worker Dentro da API

#### Descrição

Executar o processamento da Outbox dentro do mesmo processo Node responsável pelos endpoints HTTP.

Exemplo:

```text
src/server.ts

API HTTP
+
Loop de processamento dos webhooks
```

#### Benefícios

- Menor quantidade de executáveis.
- Menor complexidade de deploy.
- Menos processos para monitorar.

#### Motivos para descarte

- Reinicializações da API interrompem o processamento dos webhooks.
- Acoplamento entre responsabilidades distintas.
- Consumo de recursos do worker afeta diretamente a API.
- Dificulta escalabilidade futura.

#### Resultado

Alternativa descartada.

---

### Alternativa 2 — Processamento Síncrono Durante a Mudança de Status

#### Descrição

Executar a entrega HTTP do webhook no momento da alteração do pedido.

```text
changeStatus()
    ↓
POST webhook
```

#### Benefícios

- Menor quantidade de componentes.
- Ausência de worker dedicado.

#### Motivos para descarte

- Dependência da disponibilidade do cliente.
- Impacto direto no tempo de resposta da operação.
- Possibilidade de travar processamento de pedidos.
- Alto acoplamento entre regra de negócio e integração externa.

#### Resultado

Alternativa descartada.

---

### Alternativa 3 — Worker Acionado por Mecanismo Externo

#### Descrição

Utilizar eventos externos, filas ou mecanismos de notificação para acionar o processamento ao invés de manter um processo permanente.

#### Benefícios

- Menor atividade contínua.
- Possível redução de polling.

#### Motivos para descarte

- Exigiria componentes adicionais.
- Aumentaria complexidade operacional.
- Não apresenta benefício relevante frente ao volume esperado da feature.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Separação clara de responsabilidades

A API permanece focada em regras de negócio.

O worker permanece focado em processamento de eventos.

```text
API → Produz eventos
Worker → Consome eventos
```

---

#### Maior resiliência operacional

Falhas da API não interrompem necessariamente o processamento de webhooks.

O worker pode ser reiniciado independentemente da aplicação principal.

---

#### Escalabilidade futura

A arquitetura permite evoluções futuras como:

- Múltiplos workers.
- Particionamento do processamento.
- Processamento distribuído.

Sem necessidade de alterar o fluxo de geração dos eventos.

---

#### Reaproveitamento da plataforma existente

O worker utiliza os mesmos padrões já adotados na aplicação:

```text
Prisma
Pino
AppError
Estrutura de módulos
```

Isso reduz curva de aprendizado e custo de manutenção.

---

### Consequências Negativas

#### Processo adicional para operar

A solução passa a ter mais um componente executável.

Será necessário:

- Configurar deploy.
- Monitorar execução.
- Reiniciar em falhas.
- Coletar logs.

---

#### Maior complexidade arquitetural

Comparado a uma chamada HTTP direta, existem agora:

- Outbox
- Worker
- Retry
- DLQ
- Estados de processamento

---

#### Dependência de observabilidade

Problemas no worker podem passar despercebidos caso não exista monitoramento adequado.

A equipe precisará acompanhar:

- Filas pendentes.
- Tempo de processamento.
- Eventos em erro.
- Crescimento da Outbox.

---

## Trade-off Explícito

A decisão prioriza:

```text
Resiliência
Escalabilidade
Separação de responsabilidades
```

em detrimento de:

```text
Menor número de componentes
Menor simplicidade operacional inicial
```

Aceitamos operar um processo adicional para evitar que integrações externas impactem diretamente o fluxo crítico de pedidos.

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões arquiteturais:

- Background Worker Pattern
- Asynchronous Processing
- Transactional Outbox Consumer
- Separation of Concerns
- Producer/Consumer Pattern