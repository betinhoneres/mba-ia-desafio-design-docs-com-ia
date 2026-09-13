# Documentação do Processo — Design Docs IA para OMS Webhooks

## Sobre o desafio
Esta entrega compila um pacote completo de design docs para o Sistema de Webhooks de Notificação de Pedidos do Order Management System (OMS). O objetivo foi, a partir apenas da transcrição de uma reunião técnica e do código existente, produzir documentação de produto, arquitetura e implementação, 100% rastreável à fonte, com auxílio intensivo de IA. Todo o processo simula a atuação de um arquiteto/orquestrador de documentação apoiado por LLMs.

## Ferramentas de IA utilizadas
- **OpenAI GPT-4o (via extensão Continue):** para geração, revisão e refinamento dos documentos, entendimento do contexto das decisões e produção de exemplos.
- **Extensões de navegação em código:** busca por padrões, glob, leitura de árvores de arquivos e para localizar os pontos de integração técnico no código base.

## Workflow adotado
1. Leitura integral da transcrição para mapear decisões, requisitos e estruturas do negócio.
2. Geração das **ADRs** principais para consolidar decisões técnicas.
3. Criação da **RFC** baseada nas ADRs, alternativas consideradas e dúvidas técnicas.
4. Elaboração do **FDD** detalhando fluxos, contratos, APIs, erros e integração com arquivos reais do código.
5. Formulação do **PRD** após amadurecimento dos demais documentos, trazendo foco em produto, métricas e riscos.
6. Construção do **Tracker** de rastreabilidade, referenciando todos os requisitos e decisões a trechos exatos da transcrição ou pontos do código.
7. Iterações finais, checagens cruzadas e redação deste README.

## Prompts customizados

```plaintext
Extraia, da transcrição, pelo menos 8 requisitos funcionais explicitamente debatidos relacionados ao Sistema de Webhooks. Liste itens descartados ou adiados. Identifique também riscos discutidos, sempre referenciando timestamp e participante.
```
```plaintext
Com base na transcrição e no código dos módulos (ex: src/modules/orders/order.service.ts e prisma/schema.prisma), escreva uma ADR no formato MADR documentando a decisão sobre padrão Outbox. Inclua caminhos reais de arquivos e explique o impacto na transação de update de status.
```

## Iterações e ajustes
- Ajustei prompts para separar corretamente altura de documentos (PRD ≠ RFC ≠ FDD), pois a IA inicialmente duplicava detalhes de implementação ou requisitos.
- No Tracker, exigi referência real (timestamp/arquivo) após a IA gerar respostas superficiais e não rastreáveis, melhorando a qualidade da rastreabilidade.
- Repeti a geração dos contratos públicos até obter endpoints e exemplos em padrão JSON em sintonia com o código e limitações reais.
- Em matrizes de erros e seções de integração, reforcei prompts para garantir que só fossem citadas entidades/caminhos verdadeiros do projeto.

## Como navegar a entrega
- **docs/PRD.md:** contexto de produto, objetivos, métricas e requisitos.
- **docs/RFC.md:** proposta técnica de arquitetura.
- **docs/adrs/**: decisões arquiteturais detalhadas, uma por arquivo.
- **docs/FDD.md:** como implementar, integrações, fluxos técnicos, APIs.
- **docs/TRACKER.md:** tabela de rastreabilidade documento↔transcrição/código.
- **src/** e **prisma/**: referência dos pontos de integração (não alterados na entrega).

> Sugestão de leitura: PRD → RFC → ADRs → FDD → TRACKER.

---

Caso precise, o enunciado original do desafio está disponível no histórico deste arquivo ou diretamente no repositório base.

---
