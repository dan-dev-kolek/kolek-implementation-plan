````md
# 🧩 Plano de Implementação – Ajuste de Horário de Envio por Fuso Horário (Clientes Fora de Brasília)

**Repositório:** `kolek-api`  
**Contexto:** Atualizar o horário de disparo de mensagens automáticas considerando o fuso horário do estado do cliente.

---

## 🎯 Objetivo

Garantir que os disparos de mensagens automáticas (régua de cobrança) ocorram no mesmo horário relativo local (10h horário de Brasília), ajustando conforme o fuso horário de cada estado.

Além disso, novos clientes devem automaticamente herdar o horário correto com base no estado informado no endereço (`address.state`).

---

## 🧠 Contexto Atual

Hoje o sistema define o início dos disparos às **08h**, fixos para todos os clientes.  
Como alguns estados possuem fusos diferentes de Brasília, isso causa:

- Disparos **adiantados** em estados com fuso menor.
- Envio de mensagens **antes do horário comercial local**.
- Risco de cobrança indevida em horários inadequados.

---

## ⚙️ Regras de Ajuste

### Tabela de Fuso Horário

| Estado | Fuso oficial | Offset UTC | Diferença vs Brasília | Novo horário de disparo |
|---------|---------------|-------------|------------------------|--------------------------|
| Acre (AC) | ACT | UTC-5 | -2h | **12h** |
| Amazonas (AM, sudoeste) | ACT | UTC-5 | -2h | **12h** |
| Amazonas (AM, maior parte) | AMT | UTC-4 | -1h | **11h** |
| Rondônia (RO) | AMT | UTC-4 | -1h | **11h** |
| Roraima (RR) | AMT | UTC-4 | -1h | **11h** |
| Mato Grosso (MT) | AMT | UTC-4 | -1h | **11h** |
| Mato Grosso do Sul (MS) | AMT | UTC-4 | -1h | **11h** |
| **Demais estados** | BRT | UTC-3 | 0h | **10h** |

---

## 🧩 Implementação Técnica

### 1️⃣ Atualização do Schema Prisma

Adicionar campo opcional `timezone` na tabela `address`:

```prisma
model address {
  id                Int                    @id @default(autoincrement())
  reference_id      String                 @db.VarChar(50)
  reference_type    address_reference_type
  city_name         String                 @db.VarChar(100)
  postal_code       String                 @db.VarChar(20)
  street_type       String                 @db.VarChar(50)
  street_name       String                 @db.VarChar(50)
  neighborhood_type String                 @db.VarChar(50)
  city_code         String                 @db.VarChar(20)
  complement        String?                @db.VarChar(100)
  state             String                 @db.VarChar(2)
  number            String                 @db.VarChar(10)
  neighborhood      String                 @db.VarChar(30)
  timezone          String?                @db.VarChar(20) // Novo campo
  created_at        DateTime               @default(now())
  updated_at        DateTime               @updatedAt
  business          business_accounts?     @relation(fields: [reference_id], references: [uuid], map: "address_business_acount_map_key")
  client            client_lists?          @relation(fields: [reference_id], references: [uuid], map: "address_client_map_key")
  user              users?                 @relation(fields: [reference_id], references: [uuid], map: "address_users_map_key")

  @@unique([reference_id, reference_type])
  @@index([reference_id])
}
````

---

### 2️⃣ Script de Cadastro de Fuso Horário (migração de dados)

Criar script:

```
src/scripts/timezone/register_business_timezones.ts
```

O script deve:

1. Buscar todos os `business_accounts` com `address` cadastrado.
2. Para cada `address.state`, preencher o campo `timezone` conforme o mapeamento abaixo.
3. Atualizar o campo `timezone` via Prisma.

```ts
const TIMEZONE_MAP: Record<string, { tz: string; offset: number }> = {
  AC: { tz: 'America/Rio_Branco', offset: -2 },
  AM: { tz: 'America/Manaus', offset: -1 },
  RO: { tz: 'America/Porto_Velho', offset: -1 },
  RR: { tz: 'America/Boa_Vista', offset: -1 },
  MT: { tz: 'America/Cuiaba', offset: -1 },
  MS: { tz: 'America/Campo_Grande', offset: -1 },
};

for (const business of businesses) {
  const state = business.address?.state;
  const tz = TIMEZONE_MAP[state]?.tz || 'America/Sao_Paulo';
  await prisma.address.update({
    where: { id: business.address.id },
    data: { timezone: tz },
  });
}
```

---

### 3️⃣ Ajuste do Horário de Envio de Mensagens

#### a) Contexto

Ajustar o serviço responsável por definir a **hora inicial de disparo** (normalmente dentro de `lifecycle-scheduler` ou `lifecycle-send-notification.service.ts`).

#### b) Lógica de cálculo

Criar função utilitária em:

```
src/common/utils/timezone-dispatch.ts
```

```ts
export function getDispatchStartHourByState(state: string): number {
  const offsetMap: Record<string, number> = {
    AC: -2,
    AM: -1,
    RO: -1,
    RR: -1,
    MT: -1,
    MS: -1,
  };

  const offset = offsetMap[state] || 0;
  return 10 - offset; // 10h Brasília → ajusta para hora local
}
```

#### c) Aplicação

No momento de definir a hora inicial de disparo para o `business`:

```ts
const startHour = getDispatchStartHourByState(business.address.state);
const startTime = dayjs().set('hour', startHour).set('minute', 0).toDate();
```

---

### 4️⃣ Serviço para Novos Clientes

Ao criar um novo `business`, deve-se:

1. Detectar o estado (`address.state`).
2. Preencher automaticamente o campo `timezone` e o horário de disparo padrão.
3. Salvar em `business_settings` (ou entidade equivalente) o `dispatch_start_hour`.

---

## 🧪 Testes e Validação

| Cenário                  | Estado | Esperado                                                |
| ------------------------ | ------ | ------------------------------------------------------- |
| Cliente SP               | BRT    | Disparo 10h                                             |
| Cliente AM (maior parte) | AM     | Disparo 11h                                             |
| Cliente AC               | AC     | Disparo 12h                                             |
| Cliente RO               | RO     | Disparo 11h                                             |
| Cliente MT               | MT     | Disparo 11h                                             |
| Novo cliente RR          | RR     | `timezone` preenchido `America/Boa_Vista` e disparo 11h |

---

## 🧾 Logs e Auditoria

Registrar logs de atualização em batch:

```
✅ Business 123 - timezone America/Manaus - horário ajustado 11h
✅ Business 456 - timezone America/Rio_Branco - horário ajustado 12h
```

---

## ⚠️ Cuidados

* **Dry-run** inicial para validação antes de alterar horários em produção.
* Clientes sem endereço não devem ser processados.
* Evitar alterar manualmente horários já customizados por usuários avançados.

---

## 🚀 Deploy e Execução

### QA:

```bash
yarn ts-node src/scripts/timezone/register_business_timezones.ts
```

### Produção:

```bash
AWS_PROFILE=kolek-prod \
yarn ts-node src/scripts/timezone/register_business_timezones.ts
```

---

## ✅ Resultado Esperado

* Todos os clientes passam a ter horários ajustados conforme o fuso horário local.
* Novos clientes recebem automaticamente o horário correto.
* Redução de disparos em horários inadequados e aumento da precisão das comunicações automáticas.

---

**Status:** Pronto para desenvolvimento
**Data:** 07/11/2025
**Versão:** `v1.0.0`

```


```
