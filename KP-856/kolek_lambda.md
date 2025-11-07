````md
# 🧩 Plano de Implementação – Intervalo Configurável entre Mensagens Automáticas

**Repositório:** `kolek-lambda`  
**Módulo:** `whatsapp-sender`  
**Contexto:** Ajustar o tempo de cooldown entre mensagens automáticas com base na configuração enviada pelo backend (`send_interval_min`).

---

## 🎯 Objetivo

Implementar o uso dinâmico do campo `send_interval_min` enviado pelo backend, permitindo que cada empresa (`business_id`) tenha um intervalo configurável entre mensagens automáticas.

O valor será utilizado para determinar o cooldown antes de liberar o próximo envio de uma mesma `appKey`, substituindo o tempo fixo atualmente definido por provider.

---

## ⚙️ Motivação

Atualmente o cooldown entre mensagens automáticas é fixo (15 segundos para padrão, 4 minutos para Zappy Legacy).  
Com a nova configuração, o backend (API Kolek) envia um valor `send_interval_min` personalizado conforme tipo de integração, armazenado na tabela `whatsapp_integration_config`.

A Lambda `whatsapp-sender` deve passar a respeitar esse valor ao processar a fila Redis.

---

## 🧩 Escopo das Alterações

### 1️⃣ Atualizar o tipo `WhatsappSendingPayload`

**Arquivo:** `whatsapp-sender/types/message.ts`

Adicionar o novo campo recebido do backend:

```ts
export type WhatsappSendingPayload = {
  ...
  /** Intervalo configurado em minutos entre mensagens automáticas */
  send_interval_min?: number; // opcional para retrocompatibilidade
};
````

---

### 2️⃣ Atualizar lógica de cooldown na Lambda

**Arquivo:** `whatsapp-sender/actions/process-redis-queue.ts`

* Substituir a função atual `getCooldownSecondsByProvider()` por uma nova função `getCooldownSeconds(payload)`.
* Essa função deve calcular o cooldown em milissegundos conforme a seguinte prioridade:

| Prioridade | Condição                                 | Intervalo aplicado                      |
| ---------- | ---------------------------------------- | --------------------------------------- |
| 1️⃣        | `payload.send_interval_min` entre 1 e 20 | `payload.send_interval_min * 60 * 1000` |
| 2️⃣        | `provider === 'ZAPPY_LEGACY'`            | 4 minutos                               |
| 3️⃣        | `business_id` em `CUSTOM_BUSINESS`       | 4 minutos                               |
| 4️⃣        | Caso contrário                           | 15 segundos (valor padrão)              |

**Nova função:**

```ts
const getCooldownSeconds = (payload: WhatsappSendingPayload): number => {
  if (payload.send_interval_min && payload.send_interval_min >= 1 && payload.send_interval_min <= 20) {
    return payload.send_interval_min * MINUTE;
  }

  if (payload.provider === 'ZAPPY_LEGACY') {
    return ZAPPY_LEGACY_COOLDOWN_SECONDS;
  }

  if (payload.business_id && CUSTOM_BUSINESS.includes(payload.business_id)) {
    return ZAPPY_LEGACY_COOLDOWN_SECONDS;
  }

  return COOLDOWN_SECONDS;
};
```

---

### 3️⃣ Aplicar novo cooldown no loop principal

No trecho que processa o payload da fila:

```diff
payload = JSON.parse(rawPayload);
-cooldownSeconds =
-  getCooldownSecondsByProvider(payload.provider, payload.business_id) || COOLDOWN_SECONDS;
+cooldownSeconds = getCooldownSeconds(payload);
+lambdaConsoleLog(
+  'cooldown',
+  'log',
+  `Cooldown definido para ${cooldownSeconds / MINUTE} min (business_id=${payload.business_id}, provider=${payload.provider})`
+);
```

---

## 🧠 Regras e Validações

* `send_interval_min` deve ser **considerado apenas se for válido (entre 1 e 20)**.
* Payloads antigos (sem o campo) continuarão funcionando normalmente.
* Logs de execução devem registrar o cooldown aplicado:

  ```
  [cooldown] Cooldown definido para 4 min (business_id=xxx, provider=ZAPPY_LEGACY)
  ```

---

## 🧪 Testes de Validação

| Cenário                             | Entrada                 | Resultado Esperado                     |
| ----------------------------------- | ----------------------- | -------------------------------------- |
| Payload com `send_interval_min = 4` | provider qualquer       | Cooldown = 4 minutos                   |
| Payload com valor inválido (`25`)   | Ignorado                | Fallback padrão aplicado               |
| Provider `ZAPPY_LEGACY`             | Sem `send_interval_min` | Cooldown = 4 minutos                   |
| Business custom listado             | Sem `send_interval_min` | Cooldown = 4 minutos                   |
| Nenhuma condição atendida           | provider padrão         | Cooldown = 15 segundos                 |
| Log de execução                     | Sempre presente         | Indica cooldown aplicado e business_id |

---

## 🔍 Observabilidade

* Todos os logs são emitidos via `lambdaConsoleLog` e enviados ao CloudWatch.
* Pode ser monitorado com filtro:

  ```
  fields @timestamp, @message
  | filter @message like /Cooldown definido/
  ```

---

## 📦 Deploy Steps

1. Criar branch a partir de `develop`:

   ```
   git checkout -b feat/send-interval-config
   ```
2. Aplicar alterações conforme plano.
3. Commit:

   ```
   feat(sender): utilizar send_interval_min configurável via backend
   ```
4. Deploy no ambiente **QA**.
5. Validar logs de cooldown aplicados com diferentes configurações.

---

## ✅ Resultado Esperado

* Lambda passa a usar o intervalo enviado pelo backend (`send_interval_min`) ao processar mensagens.
* Reduz risco de bloqueios por excesso de mensagens.
* Retrocompatível com payloads antigos.
* Nenhum impacto em outras integrações.

---

**Data:** 07/11/2025
**Status:** Pronto para desenvolvimento
**Versão prevista:** `v1.0.0-beta`

```

---
```
