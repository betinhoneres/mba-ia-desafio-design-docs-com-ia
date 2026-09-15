# README

# Sobre o desafio

O objetivo deste desafio foi analisar uma transcrição de reunião técnica e, a partir dela, produzir um conjunto completo de artefatos de documentação de engenharia para a feature **Sistema de Webhooks de Notificação de Pedidos**.

A entrega foi estruturada seguindo uma abordagem de rastreabilidade completa, partindo da transcrição como fonte única da verdade. Foram produzidos documentos em diferentes níveis de abstração (PRD, RFC, ADRs, FDD e Tracker), garantindo que decisões, requisitos, restrições e contratos fossem documentados de forma consistente e posteriormente associados às suas respectivas origens.

---

# Ferramentas de IA utilizadas

## Gemini

Utilizado inicialmente para estruturar a estratégia de execução do desafio.

Responsabilidades:

- Sugerir uma metodologia para abordar o problema.
- Auxiliar na definição da sequência de produção dos documentos.
- Sugerir modelos de prompts.
- Recomendar a instalação da extensão do ChatGPT no VS Code para análise local do código.

---

## ChatGPT Mini

Utilizado como principal ferramenta para análise da transcrição e geração dos artefatos.

Responsabilidades:

- Extração de requisitos.
- Criação das ADRs.
- Criação do RFC.
- Criação do FDD.
- Criação do PRD.
- Criação do Tracker de Rastreabilidade.
- Revisões e refinamentos dos documentos.

---

## Microsoft Copilot

Utilizado durante a etapa de revisão.

Responsabilidades:

- Validação dos critérios de aceite.
- Revisão manual dos documentos.
- Verificação de consistência entre os artefatos.
- Sugestões de melhoria na organização dos documentos.
- Apoio na validação final da entrega.

---

# Workflow adotado

A execução foi organizada em etapas sequenciais para reduzir inconsistências entre documentos.

## Etapa 1 — Entendimento do desafio

Inicialmente foi realizada uma conversa com o Gemini para estruturar a abordagem de execução e definir a ordem dos artefatos.

Também foi recomendada a instalação de uma extensão no VS Code para permitir que o ChatGPT tivesse acesso ao material local do projeto.

---

## Etapa 2 — Validação do acesso à base do desafio

Após configurar a extensão, foi executado um prompt simples para validar a leitura da transcrição.

Prompt utilizado:

```text
@code Olhe o arquivo TRANSCRICAO.md e me diga quais são os principais participantes e qual é o tema central da reunião.
```

Após a resposta correta, foi considerado que a IA estava lendo corretamente o conteúdo disponibilizado.

---

## Etapa 3 — Produção dos documentos

Os documentos foram produzidos em ordem arquitetural, do mais estratégico para o mais detalhado:

```text
ADRs
↓
RFC
↓
FDD
↓
PRD
↓
Tracker de Rastreabilidade
↓
README
```

A criação nessa ordem permitiu utilizar as decisões arquiteturais como base para os documentos posteriores.

---

## Etapa 4 — Revisão e validação

Após a geração dos documentos:

- Foi realizada leitura manual de todos os arquivos.
- Foram identificadas decisões sem rastreabilidade.
- Foram removidas convenções inventadas pela IA.
- Foi realizada conferência dos critérios de aceite.
- Foi produzido o Tracker de Rastreabilidade completo.

---

## Etapa 5 — Ajustes finais

Foi feita uma verificação cruzada entre:

```text
PRD
RFC
ADRs
FDD
Tracker
```

para garantir consistência terminológica e alinhamento com a transcrição original.

---

# Prompts customizados

## Prompt 1 — Validação da leitura da transcrição

```text
@code Olhe o arquivo TRANSCRICAO.md e me diga quais são os principais participantes e qual é o tema central da reunião.
```

Objetivo:

- Validar o acesso ao conteúdo local.
- Confirmar que a IA estava analisando o material correto.

---

## Prompt 2 — Geração do RFC

```text
Produza um RFC da feature. O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto.

O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD.
```

Objetivo:

- Separar claramente arquitetura e implementação.
- Evitar mistura de responsabilidades entre RFC e FDD.

---

## Prompt 3 — Geração do FDD

```text
Produza o arquivo no formato md detalhando o "como implementar" da feature.

O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar.
```

Objetivo:

- Transformar decisões arquiteturais em um plano de implementação executável.

---

## Prompt 4 — Criação do Tracker de Rastreabilidade

```text
Produza um Tracker de Rastreabilidade em formato markdown.

Cada requisito, decisão, restrição ou contrato deve possuir uma referência explícita para sua origem na transcrição ou nos arquivos mencionados.
```

Objetivo:

- Garantir rastreabilidade.
- Reduzir alucinações da IA.
- Validar a coerência entre documentos.

---

# Iterações e ajustes

Durante a produção ocorreram várias iterações relevantes.

## Iteração 1 — Remoção de numerações inventadas

Problema encontrado:

A IA gerou identificadores como:

```text
RFC-012
ADR-021
ADR-022
ADR-023
```

Essas numerações não estavam presentes na transcrição.

Ajuste realizado:

- Remoção das numerações arbitrárias.
- Substituição por identificadores neutros.
- Utilização apenas de informações efetivamente documentadas.

---

## Iteração 2 — Validação do PRD contra a transcrição

Problema encontrado:

Alguns requisitos foram inferidos pela IA e não estavam explicitamente presentes na reunião.

Exemplo:

```text
Disponibilidade mínima
Metas operacionais específicas
Alguns requisitos de ativação/desativação
```

Ajuste realizado:

- Conferência linha a linha contra a transcrição.
- Identificação do trecho exato que originou cada requisito.
- Remoção ou ajuste dos itens sem respaldo explícito.

---

## Iteração 3 — Correção do Tracker

Problema encontrado:

A primeira versão do Tracker não atendia integralmente aos critérios de aceite.

Faltavam:

- Entradas com Fonte = CODIGO.
- Critérios de aceite rastreados.
- Integrações com arquivos citados na reunião.

Ajuste realizado:

- Inclusão de referências para arquivos mencionados.
- Inclusão dos critérios de aceite.
- Ampliação da cobertura do tracker.

---

## Iteração 4 — Revisão das ADRs

Problema encontrado:

Algumas ADRs estavam excessivamente genéricas e não referenciavam explicitamente elementos da arquitetura existente.

Ajuste realizado:

- Inclusão de módulos.
- Inclusão de caminhos de arquivos.
- Inclusão de componentes reutilizados.
- Explicitação dos trade-offs arquiteturais.

---

# Como navegar a entrega

## Ordem recomendada de leitura

### 1. PRD

Documento orientado ao produto.

```text
docs/PRD.md
```

Contém:

- Problema.
- Motivação.
- Requisitos.
- Objetivos.
- Escopo.

---

### 2. RFC

Documento arquitetural.

```text
docs/RFC.md
```

Contém:

- Contexto.
- Proposta técnica.
- Alternativas consideradas.
- Questões em aberto.

---

### 3. ADRs

Documentação das decisões arquiteturais.

```text
docs/adrs/
```

Arquivos:

```text
ADR-Transactional-Outbox.md
ADR-At-Least-Once.md
ADR-Worker-Dedicado.md
ADR-HMAC-SHA256.md
ADR-DLQ.md
ADR-Polling.md
```

---

### 4. FDD

Documento de implementação.

```text
docs/FDD.md
```

Contém:

- Fluxos técnicos.
- Contratos HTTP.
- Erros.
- Observabilidade.
- Integrações.
- Critérios de aceite.

---

### 5. Tracker de Rastreabilidade

```text
docs/Tracker.md
```

Contém:

- Referências cruzadas entre todos os documentos.
- Origem de cada requisito e decisão.
- Evidências de rastreabilidade.

---

### 6. README

```text
README.md
```

Documento de contexto desta entrega.

---

# Resumo da Entrega

Os seguintes artefatos foram produzidos:

```text
README.md

docs/
├── PRD.md
├── RFC.md
├── FDD.md
├── Tracker.md
└── adrs/
    ├── ADR-001—adocao-transactional-outbox.md
    ├── ADR-002—entrega-webhooks-at-least-once.md
    ├── ADR-003—processamento-assíncrono-worker-dedicado.md
    ├── ADR-004—assinatura-webhooks-hmac-sha256.md
    ├── ADR-005—dead-letter-queue.md
    └── ADR-006—polling-banco-dados-processamento-outbox.md
```

Todos os documentos foram produzidos a partir da transcrição fornecida e posteriormente revisados para garantir rastreabilidade, consistência e aderência aos critérios de aceite do desafio.