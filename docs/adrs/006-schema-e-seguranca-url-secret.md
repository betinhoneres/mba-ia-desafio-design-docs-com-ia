# ADR 006: Requisitos de Segurança e Validação de URL dos Endpoints de Webhook

## Status
Decidido

## Contexto
A exposição de chamadas webhooks para URLs registradas por terceiros representa risco caso não sejam seguidos padrões mínimos de validação e segurança.

## Decisão
Aceitar apenas URLs HTTPS (TLS obrigatório) para endpoints cadastrados. URLs HTTP (sem criptografia) serão rejeitadas na validação dos schemas (`zod`). Cada endpoint terá sua própria secret, que precisará ser rotacionável via endpoint API, e a secret antiga permanecerá válida por até 24h durante o período de transição. Limite de payload de 64KB — eventos fora desse limite resultam em erro, sem envio.

## Alternativas Consideradas
- Aceitar HTTP e HTTPS (risco alto de interceptação e exposição de dados sensíveis).
- Apenas uma secret global (menor controle e maior risco em caso de vazamento).
- Não impor limite de payload (risco de consumo excessivo de banda, DoS acidental).

## Consequências
- Alinha-se a práticas modernas de APIs seguras.
- Reduz vetores de ataque e vazamento de informações.
- Validação via `zod`, lógica nos módulos de webhooks e rotas de configuração em `src/modules/webhooks`.
