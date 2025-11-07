````md
# 🧩 Plano de Implementação – Intervalo de Envio de Mensagens (Régua de Cobrança)

## 🎯 Objetivo
Permitir configurar o **intervalo de tempo entre envios automáticos de mensagens** dentro da régua de cobrança, controlando a cadência e evitando bloqueios pela Meta.

---

## ⚙️ Estrutura Técnica

### Nova Tabela: `whatsapp_integration_config`

```prisma
model WhatsappIntegrationConfig {
  id                 String   @id @default(uuid())
  business_id        String   @unique
  send_interval_min  Int      @default(1)
  created_at         DateTime @default(now())
  updated_at         DateTime @updatedAt

  @@check(send_interval_min >= 1 && send_interval_min <= 20, name: "check_send_interval_range")
}
````

---

## 🧠 Regras de Negócio

| Regra              | Descrição                                                                       |
| ------------------ | ------------------------------------------------------------------------------- |
| Faixa permitida    | Valor entre **1 e 20 minutos**.                                                 |
| Default            | API Oficial → 1 min / Zappy Contábil → 4 min / Poucos lembretes → 15 min        |
| Obrigatório        | Campo obrigatório para gravação.                                                |
| Visibilidade       | Exclusivo para perfil **QA – Configuração Interna**.                            |
| Logs               | Toda operação deve ser registrada via `Logger` (sem tabela de auditoria).       |
| Efeito operacional | Valor será **enviado junto com a mensagem** ao Redis → Lambda aplicará o delay. |

---

## 🧱 Backend

### Endpoint

* `GET /business/:businessId/whatsapp-config`
* `POST /business/:businessId/whatsapp-config`

Validação:

```ts
if (send_interval_min < 1 || send_interval_min > 20) {
  throw new BadRequestException('Intervalo deve estar entre 1 e 20 minutos');
}
```

---

## 🔄 Integração no Fluxo de Envio

### 1️⃣ Ajustar tipo `NotificationToPush`

Arquivo: `src/lifecycle/types/notification-to-push.ts`

```ts
import { IntegrationData } from 'src/aws/dynamodb/entities/whatsapp-notification.entity';

interface Placeholders {
  [key: string]: string | number | boolean;
}

export interface NotificationToPush {
  notification_uuid: string;
  authkey: string;
  appkey: string;
  provider: string;
  zappy_integration_data: IntegrationData;
  zappy_legacy_token: string;
  zappy_legacy_department: string;
  zappy_legacy_department_id: number;
  zappy_legacy_api_url: string;
  zappy_legacy_instance: number | null;
  whatsapp_oficial_token: string;
  whatsapp_oficial_phone_number_id: string;
  gupshup_app_id: string | null;
  gupshup_app_name: string | null;
  gupshup_app_api_token: string | null;
  sending_flow: string;
  to: string;
  template: string;
  placeholders: Placeholders;
  positional_placeholders: string[];
  business_id: string;
  /** Intervalo (em minutos) entre mensagens automáticas — aplicado pela Lambda */
  send_interval_min: number;
}
```

---

### 2️⃣ Atualizar `lifecycle-send-notification.service.ts`

Arquivo: `src/lifecycle/lifecycle-send-notification.service.ts`

Na etapa onde o serviço **gera o payload para enfileirar no Redis**, adicione a leitura da configuração e o campo `send_interval_min` ao payload.

#### Exemplo:

```ts
const config = await this.prisma.whatsappIntegrationConfig.findUnique({
  where: { business_id },
});

let sendInterval = config?.send_interval_min;

// Caso não exista configuração, aplicar defaults conforme integração
if (!sendInterval) {
  if (provider === 'WHATSAPP_OFICIAL') sendInterval = 1;
  else if (provider === 'ZAPPY') sendInterval = 4;
  else sendInterval = 15;
}

this.logger.log(
  `[LifecycleNotification] Intervalo configurado: ${sendInterval}min para business_id=${business_id}`
);

// Adicionar o intervalo no payload que será enviado para a fila do Redis
const payload: NotificationToPush = {
  notification_uuid,
  authkey,
  appkey,
  provider,
  zappy_integration_data,
  zappy_legacy_token,
  zappy_legacy_department,
  zappy_legacy_department_id,
  zappy_legacy_api_url,
  zappy_legacy_instance,
  whatsapp_oficial_token,
  whatsapp_oficial_phone_number_id,
  gupshup_app_id,
  gupshup_app_name,
  gupshup_app_api_token,
  sending_flow,
  to,
  template,
  placeholders,
  positional_placeholders,
  business_id,
  send_interval_min: sendInterval, // 👈 novo campo incluído
};

// Enviar payload normalmente para a fila do Redis
await this.redisQueue.publish(payload);
```

---

## 🧾 Logs de Auditoria

* Nenhuma tabela extra necessária.
* Cada envio deve registrar no log:

  ```
  [LifecycleNotification] Intervalo configurado: 4min (provider=ZAPPY, business_id=xxxx)
  ```
* Cada alteração de configuração via endpoint também deve gerar log:

  ```
  [WhatsappIntegrationConfig] Alteração de intervalo: 2 → 5 (business_id=xxxx, user=admin@kolek)
  ```

---

## 🧩 Frontend

* Slider 1–20 minutos (passo 1).
* Valor default conforme integração.
* Texto explicativo:
  *“Este campo define o intervalo de tempo utilizado para envio das mensagens, evitando possíveis bloqueios por parte da Meta.”*
* Visível apenas para **QA – Configuração Interna**.

---

## 🧪 Testes

| Tipo       | Cenário                                             | Resultado esperado |
| ---------- | --------------------------------------------------- | ------------------ |
| Unitário   | Valor fora da faixa (ex.: 25)                       | Erro 400           |
| Unitário   | `send_interval_min` incluído no payload Redis       | ✅                  |
| Integração | Mensagens enviadas com intervalo correto via Lambda | ✅                  |
| Log        | Registrar `send_interval_min` usado                 | ✅                  |

---

## 🚀 Deploy

1. Criar migration:

   ```bash
   npx prisma migrate dev --name add_whatsapp_integration_config
   ```
2. Validar default (1–4–15) nos `business_id` existentes.
3. Deploy backend.
4. Confirmar no log:

   ```
   [LifecycleNotification] Intervalo configurado: 4min para business_id=xxxx
   ```

---

## ✅ Resultado Esperado

* O backend inclui `send_interval_min` no payload.
* O intervalo é aplicado **na Lambda de envio**.
* Logs informam o intervalo usado.
* Nenhum impacto em perfis não-QA.

```

---


```
