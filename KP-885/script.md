```md
# 🧩 Plano de Implementação – Ajuste de Tarifas QI Tech (via `set-fees-v2`)

**Repositório:** `kolek-api`  
**Módulo:** `digital-account-fees`  
**Contexto:** Corrigir as tarifas de clientes sem pacote de boletos, garantindo que o responsável pelas tarifas seja o próprio cliente e os valores estejam de acordo com a planilha fornecida.

---

## 🎯 Objetivo

Criar um **script automatizado** para atualizar tarifas de clientes na QI Tech via endpoint interno `digital-account-fees/{account_id}/set-fees-v2`, aplicando o payload com as tarifas corretas e ajustando o `billing_account_key` para a própria conta do cliente.

---

## ⚙️ Escopo da Task

### Entrada

Planilha (XLSX) contendo os campos:
| business_id | qitech_account_id | tarifa_boleto | tarifa_pix | tarifa_transferencia |
|--------------|-------------------|----------------|-------------|-----------------------|

---

## 🧠 Regras

Para cada cliente listado:
1. O campo `billing_account_key` deve ser igual ao `account_key` do cliente.
2. As tarifas devem seguir o formato do payload abaixo, com valores vindos da planilha.
3. As tarifas devem ser aplicadas via chamada HTTP interna:
```

POST /digital-account-fees/{qitech_account_id}/set-fees-v2

````

---

## 🧩 Payload de Exemplo

```json
{
"bank_slip": {
 "bank_slip_instant_registration": {
   "billing_account_key": "{account_key}",
   "amount": 1
 },
 "protest_request": {
   "billing_account_key": "{account_key}",
   "amount": 10
 },
 "payment": {
   "billing_account_key": "{account_key}",
   "amount": 1.5
 },
 "payment_qr_code": {
   "billing_account_key": "{account_key}",
   "amount": 0.6
 }
},
"pix_transfer": {
 "incoming_pix": {
   "billing_account_key": "{account_key}",
   "amount": 0.5
 },
 "outgoing_pix": {
   "billing_account_key": "{account_key}",
   "amount": 0.5
 }
}
}
````

> ⚙️ Todos os valores (`amount`) devem ser substituídos pelos da planilha.

---

## 🧩 Implementação Técnica

### 1️⃣ Local do Script

Adicionar o script em:

```
src/scripts/fix-qitech-fees/
└── fix_qitech_fees_v2.ts
```

---

### 2️⃣ Dependências

```bash
yarn add axios xlsx
```

---

### 3️⃣ Estrutura do Script

```ts
import axios from 'axios';
import * as XLSX from 'xlsx';
import * as dotenv from 'dotenv';

dotenv.config();

const workbook = XLSX.readFile('scripts/fix-qitech-fees/ajuste_tarifas.xlsx');
const sheet = workbook.Sheets[workbook.SheetNames[0]];
const rows = XLSX.utils.sheet_to_json(sheet);

const API_BASE_URL = process.env.KOLEK_API_URL || 'https://api.kolek.com.br';
const API_TOKEN = process.env.INTERNAL_API_TOKEN;

const client = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    Authorization: `Bearer ${API_TOKEN}`,
    'Content-Type': 'application/json',
  },
});

(async () => {
  for (const row of rows as any[]) {
    const { business_id, qitech_account_id, tarifa_boleto, tarifa_pix, tarifa_transferencia } = row;

    try {
      // 1️⃣ Buscar account_key do cliente
      const { data: account } = await client.get(`/digital-accounts/${qitech_account_id}`);
      const accountKey = account?.account_key;
      if (!accountKey) {
        console.error(`❌ Conta não encontrada para business ${business_id}`);
        continue;
      }

      // 2️⃣ Montar payload
      const payload = {
        bank_slip: {
          bank_slip_instant_registration: {
            billing_account_key: accountKey,
            amount: Number(tarifa_boleto),
          },
          protest_request: {
            billing_account_key: accountKey,
            amount: 10,
          },
          payment: {
            billing_account_key: accountKey,
            amount: Number(tarifa_boleto),
          },
          payment_qr_code: {
            billing_account_key: accountKey,
            amount: Number(tarifa_boleto),
          },
        },
        pix_transfer: {
          incoming_pix: {
            billing_account_key: accountKey,
            amount: Number(tarifa_pix),
          },
          outgoing_pix: {
            billing_account_key: accountKey,
            amount: Number(tarifa_transferencia),
          },
        },
      };

      // 3️⃣ Aplicar tarifas via endpoint interno
      const url = `/digital-account-fees/${qitech_account_id}/set-fees-v2`;
      await client.post(url, payload);

      console.log(`✅ Tarifas aplicadas com sucesso para business_id=${business_id}`);

    } catch (error: any) {
      console.error(`❌ Erro ao processar business_id=${row.business_id}`, error.response?.data || error.message);
    }
  }
})();
```

---

## 🔐 Configurações de Ambiente

Adicionar variáveis no `.env`:

```env
KOLEK_API_URL=https://api.kolek.com.br
INTERNAL_API_TOKEN=<token com permissão de digital-account-fees>
```

---

## 🧪 Testes e Validação

| Cenário                                 | Resultado Esperado                              |
| --------------------------------------- | ----------------------------------------------- |
| Conta com tarifas incorretas            | Payload aplicado com sucesso                    |
| Conta sem `billing_account_key` correto | Atualiza automaticamente                        |
| Conta inexistente                       | Loga erro e continua                            |
| Token inválido                          | Falha controlada (erro 401)                     |
| Reexecução                              | Idempotente (tarifas sobrescritas corretamente) |

---

## 🧾 Logs de Execução

O script deve exibir logs resumidos por cliente:

```
✅ Tarifas aplicadas para business_id=xxx qitech_account_id=yyy
❌ Erro ao processar business_id=zzz | 403 Forbidden
```

---

## 🚀 Execução

### QA:

```bash
yarn ts-node src/scripts/fix-qitech-fees/fix_qitech_fees_v2.ts
```

### Produção:

```bash
AWS_PROFILE=kolek-prod \
KOLEK_API_URL=https://api.kolek.com.br \
INTERNAL_API_TOKEN=... \
yarn ts-node src/scripts/fix-qitech-fees/fix_qitech_fees_v2.ts
```

---

## ✅ Resultado Esperado

* Todas as contas da planilha terão `billing_account_key` atualizado para a própria conta.
* As tarifas configuradas no endpoint `set-fees-v2` refletem os valores corretos.
* Logs indicam o sucesso ou falha por cliente.
* Após execução, verificação na QI Tech confirma ajustes corretos.

---

**Status:** Pronto para desenvolvimento
**Data:** 07/11/2025
**Versão:** `v2.0.0`

```

---

```
