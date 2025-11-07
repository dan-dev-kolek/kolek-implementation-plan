````md
# [NF] Permitir Nova Emissão após Cancelamento da Nota

**Repositório:** `kolek-api`  
**Módulo:** `plug-notas`  
**Serviço principal:** `src/plug-notas/plug-notas.service.ts`

---

## 🎯 Objetivo

Permitir que o usuário emita uma **nova nota fiscal** para uma **fatura que possua nota cancelada**, preservando o histórico das notas anteriores e respeitando regras de integridade e consistência fiscal.

---

## 🧠 Contexto Atual

Atualmente:
- Cada fatura (`account_receivable`) possui **apenas uma nota associada** (`nfs.account_receivable_id` é `unique`);
- Se uma nota é cancelada, o sistema **não permite reemissão**;
- O campo `idIntegracao` da requisição à PlugNotas usa o mesmo `uuid` da fatura — o que impede novas emissões, pois o ID de integração deve ser **único por emissão**.

---

## ⚙️ Nova Regra de Negócio

### Cenário principal:
- Uma fatura possui uma nota **cancelada**.
- O usuário tenta emitir uma nova nota fiscal.

### Regras:
1. O sistema deve **permitir** nova emissão apenas se **todas as notas anteriores estiverem canceladas** (`is_canceled = true`).
2. O sistema deve **gerar um novo registro de NF** associado à mesma fatura.
3. O **`idIntegracao`** enviado à PlugNotas deve ser **único** (ex.: `receivable.uuid + "-" + new Date().getTime()`).
4. A nova nota deve iniciar com status `PROCESSING`, e após sucesso, `EMITIDA`.
5. A nota cancelada permanece salva, com status `CANCELADA`.
6. A nota mais recente deve ser exibida **acima** das antigas na listagem (ordem decrescente de `created_on`).

---

## ✅ Critérios de Aceitação

| Cenário | Resultado Esperado |
|----------|--------------------|
| Fatura sem nota cancelada | Emite normalmente (fluxo atual) |
| Fatura com nota cancelada | Exibe modal de confirmação → se confirmado, cria nova NF |
| Fatura com nota ativa | Bloqueia emissão com aviso |
| Nova nota emitida | Status `Emitida`, associada à mesma fatura |
| Histórico | Mostra todas as notas (canceladas e ativas), mais recentes primeiro |

---

## 🧩 Alterações no Banco de Dados

### 1️⃣ Criar nova tabela `nfs_history`

Essa tabela armazenará todas as emissões (ativas e canceladas) associadas à mesma fatura, garantindo rastreabilidade.

```prisma
model nfs_history {
  id                    Int                 @id @default(autoincrement())
  uuid                  String              @unique(map: "nfs_history_uuid") @db.VarChar(50)
  nfs_id                Int                 @db.Int
  account_receivable_id String              @db.VarChar(50)
  status                nf_status           @default(PROCESSING)
  type                  nf_type
  response_payload      Json?
  created_on            DateTime            @default(now()) @db.Timestamp(0)
  canceled_at           DateTime?           @db.Timestamp(0)
  reason                String?             @db.Text
  attempt               Int                 @default(0)

  @@index([account_receivable_id])
  @@index([nfs_id])
}
````

> 🔎 Essa tabela servirá apenas como histórico.
> Cada nova emissão de NF (ativa ou cancelada) é registrada aqui após o `emit` ou `cancel`.

---

## 🧩 Alterações no Model Atual (`nfs`)

Remover a restrição de unicidade do campo `account_receivable_id`:

```diff
- account_receivable_id String              @unique @db.VarChar(50)
+ account_receivable_id String              @db.VarChar(50)
```

Isso permitirá **múltiplas notas** associadas a uma mesma fatura.

---

## ⚙️ Alterações no Serviço PlugNotas

Arquivo: `src/plug-notas/plug-notas.service.ts`

### 1️⃣ Ajustar geração do `idIntegracao`

```diff
return {
- idIntegracao: receivable.uuid,
+ idIntegracao: `${receivable.uuid}-${Date.now()}`, // novo identificador único
  prestador: { cpfCnpj: receivable.business.business_no },
  ...
}
```

> 💡 O ID de integração passa a ser único para cada emissão, permitindo múltiplas notas vinculadas à mesma fatura.

---

### 2️⃣ Lógica de Verificação antes de Emitir

Adicionar verificação antes da chamada de emissão:

```ts
const existingNotes = await this.prisma.nfs.findMany({
  where: { account_receivable_id: receivable.uuid },
  orderBy: { created_on: 'desc' },
});

const activeNote = existingNotes.find(
  (n) => n.status === 'EMITIDA' || n.status === 'PROCESSING',
);

if (activeNote) {
  throw new BadRequestException(
    'Já existe uma nota ativa ou em processamento para esta fatura.',
  );
}

const canceledNotes = existingNotes.filter((n) => n.is_canceled);
if (canceledNotes.length > 0) {
  // Front deve exibir modal de confirmação
  this.logger.log(
    `⚠️ Nota cancelada encontrada para fatura ${receivable.uuid}. Permitindo reemissão mediante confirmação.`,
  );
}
```

---

### 3️⃣ Gravação no Histórico

Após cada emissão bem-sucedida:

```ts
await this.prisma.nfs_history.create({
  data: {
    uuid: newNf.uuid,
    nfs_id: newNf.id,
    account_receivable_id: receivable.uuid,
    status: newNf.status,
    type: newNf.type,
    response_payload: newNf.response_payload,
    attempt: newNf.attempt,
  },
});
```

Ao cancelar uma nota, registrar também:

```ts
await this.prisma.nfs_history.create({
  data: {
    uuid: canceledNf.uuid,
    nfs_id: canceledNf.id,
    account_receivable_id: canceledNf.account_receivable_id,
    status: 'CANCELADA',
    reason: canceledNf.reason,
    canceled_at: new Date(),
  },
});
```

---

## 🧪 Fluxo Esperado

| Passo | Ação                                                   | Resultado                                 |
| ----- | ------------------------------------------------------ | ----------------------------------------- |
| 1     | Usuário clica “Emitir NF” em fatura com nota cancelada | Sistema mostra modal                      |
| 2     | Usuário confirma                                       | Nova NF é emitida com novo `idIntegracao` |
| 3     | Banco salva novo registro em `nfs` + `nfs_history`     |                                           |
| 4     | Front exibe nota nova acima da cancelada               |                                           |
| 5     | Cancelamento de nova nota também é salvo no histórico  |                                           |

---

## 🧾 Logs de Execução

```
[PlugNotasService] ⚠️ Nota cancelada encontrada para fatura 2e2c... — aguardando confirmação
[PlugNotasService] ✅ Nova NF emitida — idIntegracao: 2e2c...-1731011800799
[PlugNotasService] 🧾 Histórico atualizado (2 registros para fatura 2e2c...)
```

---

## ⚠️ Cuidados Técnicos

* **Não reaproveitar o mesmo `idIntegracao`** da nota anterior;
* Garantir que **apenas uma nota ativa** exista por fatura;
* Evitar sobrescrever notas anteriores;
* `nfs_history` deve conter registros tanto de emissão quanto de cancelamento;
* Manter integridade de foreign keys entre `nfs` e `nfs_history`.

---

## ✅ Resultado Esperado

* O sistema permite reemissão apenas se a nota anterior estiver cancelada;
* Cada emissão fica registrada no histórico (`nfs_history`);
* As notas aparecem em ordem cronológica decrescente;
* Nenhuma duplicidade ou conflito de `idIntegracao` ocorre.

---

**Status:** Pronto para desenvolvimento
**Data:** 07/11/2025
**Versão:** `v1.0.0`

```

---

```
