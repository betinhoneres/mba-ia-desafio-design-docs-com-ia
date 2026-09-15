# ADR — Assinatura de Webhooks utilizando HMAC-SHA256

## Status

Aceita

---

## Contexto

O sistema de Webhooks de Notificação de Pedidos enviará eventos para endpoints operados por clientes externos à plataforma.

Neste modelo de integração, é necessário permitir que o consumidor valide:

1. Que a requisição foi realmente enviada pela plataforma.
2. Que o payload não foi alterado durante o transporte.
3. Que endpoints distintos possuam credenciais independentes.
4. Que credenciais comprometidas possam ser substituídas sem interrupção imediata da integração.

Como os webhooks trafegam informações de pedidos para sistemas externos, existe o risco de:

- Falsificação de requisições.
- Modificação do payload.
- Vazamento de credenciais.
- Uso indevido de uma secret comprometida.

A arquitetura precisava adotar um mecanismo de autenticação amplamente suportado pelo mercado, compatível com qualquer stack utilizada pelos clientes e que pudesse ser implementado sem dependência de infraestrutura adicional.

---

## Decisão

A plataforma utilizará **HMAC-SHA256** para assinar todas as requisições de webhook enviadas aos clientes.

A assinatura será calculada sobre o corpo completo da requisição utilizando uma secret compartilhada entre a plataforma e o consumidor.

O resultado da assinatura será enviado no header:

```http
X-Signature
```

Cada configuração de webhook possuirá sua própria secret.

Não existirá secret global compartilhada entre múltiplos webhooks.

---

### Exemplo Conceitual

```text
payload
        +
secret do webhook
        ↓
HMAC-SHA256
        ↓
X-Signature
```

O consumidor utilizará a mesma secret para recalcular a assinatura localmente e comparar os resultados.

---

### Isolamento por Webhook

Cada registro de configuração armazenará individualmente:

```text
webhook_id
customer_id
url
secret
status
```

Isso garante que o vazamento de uma secret comprometa apenas um único endpoint configurado.

---

### Rotação de Secret

A plataforma suportará rotação de secret.

Durante o processo de rotação:

```text
Nova Secret
      +
Secret Anterior
```

permanecerão válidas simultaneamente durante um período de transição.

Após esse período, apenas a nova secret continuará válida.

Essa estratégia permite que clientes atualizem seus sistemas sem indisponibilidade.

---

### HTTPS Obrigatório

A assinatura HMAC não substitui TLS.

Todos os webhooks deverão utilizar:

```text
https://
```

Endpoints utilizando:

```text
http://
```

serão rejeitados durante o cadastro ou atualização da configuração.

---

### Headers Relacionados

Além da assinatura, as requisições deverão incluir os seguintes metadados:

```http
X-Event-Id
X-Signature
X-Timestamp
X-Webhook-Id
```

O objetivo é facilitar:

- Deduplicação.
- Auditoria.
- Verificação temporal.
- Identificação do endpoint configurado.

---

## Componentes Impactados

### Módulo

```text
src/modules/webhooks
```

### Camadas

```text
webhook.controller
webhook.service
webhook.repository
webhook.schemas
webhook.processor
```

### Estruturas Persistentes

```text
webhooks
webhook_outbox
webhook_deliveries
```

### Componentes de Infraestrutura Reutilizados

```text
Prisma
Zod
AppError
Pino
Middleware de erros
```

---

## Alternativas Consideradas

### Alternativa 1 — Sem Assinatura

#### Descrição

Enviar apenas requisições HTTPS sem mecanismo adicional de autenticação.

#### Benefícios

- Implementação extremamente simples.
- Menor esforço para clientes consumidores.

#### Motivos para descarte

- Consumidor não consegue validar a origem da requisição.
- Não existe garantia de integridade do payload.
- Menor nível de segurança.
- Inadequado para transferência de eventos relacionados a pedidos.

#### Resultado

Alternativa descartada.

---

### Alternativa 2 — Secret Global da Plataforma

#### Descrição

Utilizar uma única secret compartilhada por todos os webhooks cadastrados.

#### Benefícios

- Menor complexidade de gerenciamento.
- Estrutura de dados mais simples.

#### Motivos para descarte

- Vazamento de uma única secret comprometeria todos os clientes.
- Dificulta rotação seletiva.
- Não permite isolamento adequado entre integrações.

#### Resultado

Alternativa descartada.

---

### Alternativa 3 — Assinatura Assimétrica (RSA ou Certificados)

#### Descrição

Assinar eventos utilizando criptografia assimétrica.

#### Benefícios

- Maior robustez criptográfica.
- Elimina compartilhamento direto de segredos.

#### Motivos para descarte

- Complexidade operacional significativamente maior.
- Necessidade de gestão de chaves.
- Sobrecarga para clientes.
- Não apresenta benefício proporcional para o escopo atual.

#### Resultado

Alternativa descartada.

---

## Consequências

### Consequências Positivas

#### Garantia de Integridade

O consumidor consegue detectar modificações acidentais ou maliciosas no payload.

```text
Payload Alterado
        ↓
Assinatura Inválida
```

---

#### Garantia de Autenticidade

Consumidores conseguem validar que a requisição foi produzida pela plataforma.

---

#### Adoção de Padrão de Mercado

HMAC-SHA256 possui suporte nativo ou amplamente disponível em praticamente todas as linguagens utilizadas por clientes.

---

#### Isolamento entre Integrações

Cada webhook possui sua própria credencial.

O comprometimento de um endpoint não afeta os demais.

---

#### Rotação Segura

A troca de credentials pode ocorrer sem interrupção imediata das integrações.

---

### Consequências Negativas

#### Gestão de Segredos

A plataforma passa a ser responsável por:

- Geração segura.
- Armazenamento seguro.
- Rotação.
- Controle do ciclo de vida das secrets.

---

#### Maior Complexidade para Consumidores

Clientes precisam implementar lógica adicional para:

- Armazenar a secret.
- Calcular o HMAC recebido.
- Comparar assinaturas.

---

#### Necessidade de Documentação

A integração exige documentação clara contendo:

- Algoritmo utilizado.
- Processo de validação.
- Processo de rotação.
- Tratamento de falhas.

---

## Trade-off Explícito

A decisão prioriza:

```text
Segurança
Integridade
Autenticidade
Isolamento entre clientes
```

em detrimento de:

```text
Simplicidade de implementação
Ausência de gestão de segredos
```

Aceitamos a complexidade adicional de gerenciamento de secrets para garantir que consumidores possam validar a origem e a integridade dos eventos recebidos.

---

## Padrões Relacionados

Esta ADR formaliza a adoção dos seguintes padrões:

- Message Authentication Code (MAC)
- HMAC Authentication
- Shared Secret Authentication
- Secret Rotation