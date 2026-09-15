# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados
| Campo | Valor |
|---------|---------|
| Título | Sistema de Webhooks de Notificação de Pedidos |
| Autor | Larissa (Tech Lead) |
| Status | Proposed |
| Data | 2026-09-14 |
| Revisores | Larissa, Marcos, Bruno, Diego, Sofia |

## Resumo Executivo (TL;DR)

Propomos a implementação de um sistema de webhooks outbound para notificar clientes B2B sobre mudanças de status de pedidos.

A solução adotará o padrão **Transactional Outbox** utilizando a infraestrutura atual baseada em MySQL, evitando a introdução de componentes adicionais de mensageria. Eventos serão registrados na mesma transação responsável pela alteração do pedido e posteriormente processados por um worker assíncrono dedicado.

A proposta prioriza:

- Confiabilidade na geração dos eventos.
- Baixa complexidade operacional.
- Entrega assíncrona resiliente a falhas.
- Segurança baseada em HMAC-SHA256.
- Garantia de entrega *at-least-once*.
- Uso máximo da infraestrutura e padrões já existentes.

A solução atende ao requisito de notificação em tempo quase real (latência inferior a 10 segundos) sem introduzir Redis, Kafka ou outros componentes operacionais adicionais.


## Contexto e Problema

Clientes B2B estratégicos solicitaram notificações em tempo real quando o status de seus pedidos for alterado. Atualmente, essas integrações dependem de polling contínuo no endpoint de consulta de pedidos, gerando:

- Maior carga sobre a API.
- Custos operacionais para os clientes.
- Latência desnecessária na propagação das atualizações.
- Risco comercial associado à experiência de integração.

O objetivo desta RFC é definir a abordagem arquitetural para disponibilizar notificações assíncronas confiáveis, mantendo simplicidade operacional e compatibilidade com a plataforma existente.


## Proposta Técnica

### Visão Geral

A proposta é baseada em cinco componentes principais:

#### 1. Configuração de Webhooks

Clientes poderão registrar endpoints HTTPS que desejam receber notificações sobre mudanças específicas de status dos pedidos.

A configuração armazenará:

- Endpoint de destino
- Secret de assinatura
- Status monitorados
- Estado da configuração

---

#### 2. Transactional Outbox

Quando o status de um pedido for alterado, o evento será registrado em uma tabela de Outbox dentro da mesma transação utilizada pela lógica de negócio.

Essa abordagem garante consistência entre:

- Ordem atualizada
- Histórico de status
- Ajustes de estoque
- Registro do evento

Caso a transação falhe, nenhum evento será publicado.

---

#### 3. Worker Assíncrono

Um processo dedicado realizará a leitura periódica da Outbox e executará as entregas HTTP.

Características da proposta:

- Processo independente da API.
- Polling periódico.
- Processamento dos eventos pendentes.
- Atualização do estado de entrega.

O worker compartilha banco de dados e stack tecnológica, mas opera como processo separado para evitar acoplamento ao ciclo de vida da API.

---

#### 4. Entrega Confiável

As notificações seguirão semântica de entrega *at-least-once*.

Consequências da abordagem:

- Um evento pode ser entregue mais de uma vez.
- Cada evento possuirá identificador único.
- Consumidores deverão tratar deduplicação.

A escolha privilegia simplicidade e robustez em detrimento da complexidade necessária para implementar garantias *exactly-once*.

---

#### 5. Segurança

As entregas serão protegidas através de:

- HTTPS obrigatório.
- Assinatura HMAC-SHA256.
- Secret exclusiva por webhook.
- Suporte à rotação de secret.
- Inclusão de timestamp de envio.

A validação da autenticidade passa a ser responsabilidade compartilhada entre plataforma e consumidor. 

---

## Alternativas Consideradas

### Alternativa 1 — Webhook Síncrono dentro do Order Service

#### Descrição

Executar a chamada HTTP ao consumidor durante a própria transação de mudança de status do pedido.

#### Vantagens

- Arquitetura simples.
- Menor quantidade de componentes.

#### Motivos para descarte

- Introduz dependência de disponibilidade do cliente.
- Aumenta o tempo da transação principal.
- Pode bloquear a atualização de pedidos.
- Torna impossível garantir consistência sem efeitos colaterais.

#### Decisão

**Descartada.** A abordagem foi considerada inadequada para workloads críticos de pedidos.

---

### Alternativa 2 — Redis Streams / Mensageria Externa

#### Descrição

Registrar eventos em Redis Streams ou componente dedicado de mensageria.

#### Vantagens

- Melhor base para escalabilidade futura.
- Menor dependência do banco relacional para filas.

#### Motivos para descarte

- Necessidade de novos componentes operacionais.
- Maior custo de infraestrutura.
- Complexidade de manutenção incompatível com o tamanho atual da equipe.
- Benefício considerado insuficiente para o escopo presente.

#### Decisão

**Descartada.** A equipe optou por reutilizar o MySQL já existente. 【1-6fd56f】

---

### Alternativa 3 — Notificação por Trigger de Banco

#### Descrição

Utilizar triggers do banco como mecanismo de disparo do processamento.

#### Vantagens

- Menor intervalo entre criação e processamento do evento.

#### Motivos para descarte

- MySQL não oferece mecanismo nativo equivalente a LISTEN/NOTIFY.
- Exigiria soluções indiretas e pouco confiáveis para sinalizar processos externos.
- Aumentaria o acoplamento entre banco e processamento.

#### Decisão

**Descartada.** Polling simples foi considerado suficiente para atender ao SLA esperado. 【1-6fd56f】


## Questões em Aberto

Os seguintes pontos foram identificados durante a discussão, mas deliberadamente adiados para futuras decisões arquiteturais.

### Estratégia de Escalabilidade do Worker

Atualmente a proposta assume um único worker.

Ainda não foi definida a estratégia de escalabilidade para cenários futuros com múltiplos workers, incluindo:

- Garantia de ordering.
- Particionamento por pedido.
- Estratégias de locking.

A necessidade será reavaliada conforme crescimento de volume. 【1-6fd56f】

---

### Rate Limiting de Entrega

Não foi definido se o sistema limitará a quantidade de requisições enviadas para um mesmo cliente em períodos de pico.

Questão pendente:

- Devemos proteger consumidores contra rajadas de eventos?

Decisão adiada até que haja evidência de problema em produção.

---

### Notificações Proativas de Falhas

Foi discutida a possibilidade de envio de e-mails quando webhooks apresentarem falhas repetidas.

A proposta foi retirada desta fase e poderá retornar posteriormente como evolução do produto.

---

### 4. Estratégia de Retenção da Outbox

A necessidade de arquivamento e limpeza de eventos entregues foi mencionada, mas ficou explicitamente fora do escopo da feature atual.

O ciclo de vida dos registros deverá ser tratado em iniciativa futura.

---

## Impactos

### Impactos Positivos

- Redução significativa de polling.
- Melhor experiência de integração para clientes.
- Menor carga recorrente na API de pedidos.
- Maior rapidez na propagação de eventos de negócio.
- Base reutilizável para futuras notificações da plataforma.

---

## Riscos

### Crescimento da Outbox

Acúmulo excessivo de registros pode degradar consultas e processamento.

Mitigação:

- Indexação adequada.
- Leitura em lotes.
- Estratégia futura de arquivamento.

---

### Indisponibilidade de Consumidores

Falhas prolongadas podem impedir entregas.

Mitigação:

- Retry com backoff.
- DLQ.
- Replay manual.


## Decisões Relacionadas

Os seguintes ADRs deverão ser produzidos para registrar decisões permanentes derivadas desta RFC:

- ADR-001—adocao-transactional-outbox.md
- ADR-002—entrega-webhooks-at-least-once.md
- ADR-003—processamento-assíncrono-worker-dedicado.md
- ADR-004—assinatura-webhooks-hmac-sha256.md
- ADR-005—dead-letter-queue.md
- ADR-006—polling-banco-dados-processamento-outbox.md

---

# Status da RFC

**Proposed**

A RFC será considerada aprovada após revisão de:

- Tech Lead
- Engenharia de Plataforma
- Engenharia de Pedidos
- Segurança
- Product Management