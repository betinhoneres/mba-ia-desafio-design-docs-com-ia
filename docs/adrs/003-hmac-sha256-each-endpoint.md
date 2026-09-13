# ADR 003: Assinatura HMAC-SHA256 com Secret por Endpoint para Autenticação de Webhooks

## Status
Decidido

## Contexto
Webhooks expõem dados de pedidos para sistemas externos. É necessário garantir autenticidade e integridade das notificações enviadas, evitando ataques de falsificação ou interceptação/manipulação do payload. Empresas clientes esperam validar que a requisição veio do emissor legítimo.

## Decisão
Toda notificação de webhook será assinada com HMAC-SHA256 sobre o payload, usando uma secret única por endpoint cadastrado na base. O valor da assinatura será enviado no header `X-Signature`. O cliente também recebe o valor da secret por API e pode solicitar rotação, mantendo a antiga válida por 24 horas para transição segura.

## Alternativas Consideradas
- Não assinar (expondo-se a ataques).
- Usar uma única secret global (mais vulnerável a leaks; um cliente comprometido comprometeria todos).
- Outras formas de autenticação menos difundidas (ex. JWT, assinatura assimétrica — complexidade e interoperabilidade pior nos clientes).

## Consequências
- Alinha-se com o padrão do mercado (GitHub, Stripe, etc).
- Facilita integração dos clientes, que só precisam conhecer e validar HMAC-SHA256.
- Melhora a segurança; mínimo impacto na arquitetura e no fluxo existente.
- Estrutura e controle das secrets podem ficar em `src/modules/webhooks` e modelo em `prisma/schema.prisma`.
