````md
# [FN] NF: Emissão pelo Valor Efetivo (Líquido/Pago)

**Repositório:** `kolek-api`  
**Arquivo principal:** `src/plug-notas/plug-notas.service.ts`  
**Método afetado:** `buildEmitPayloadFromReceivable`

---

## 🎯 Objetivo

Ajustar a lógica de cálculo do valor emitido na NFSe para refletir **corretamente o valor líquido ou o valor efetivamente pago** da fatura, conforme a **modalidade de emissão configurada** e o status da fatura.

---

## 🧠 Contexto Atual

Hoje o método `buildEmitPayloadFromReceivable()` define o valor da nota com base diretamente em:

```ts
valor: {
  servico: Number(receivable.amount),
  descontoCondicionado: 0,
  descontoIncondicionado: 0,
},
````

Isso significa que o valor emitido **é sempre o valor bruto da fatura**, sem considerar descontos, juros, ou o valor efetivamente pago — o que causa divergências entre o valor contábil e o fiscal.

---

## ✅ Critérios de Aceitação

### Regras gerais

| Tipo de Emissão                               | Regra                        | Valor da Nota                               |
| --------------------------------------------- | ---------------------------- | ------------------------------------------- |
| **Automática (ao gerar fatura)**              | Sem pagamento confirmado     | = `valor_bruto` da fatura                   |
| **Pós-pagamento (após pagamento confirmado)** | Após registro do pagamento   | = `valor_pago` (com juros/desconto efetivo) |
| **Sob demanda (manual)**                      | Se a fatura estiver paga     | = `valor_pago`                              |
|                                               | Se a fatura não estiver paga | = `valor_bruto`                             |

---

### Regras para descontos e acréscimos

| Tipo                     | Quando aplicar                               | Campo                                         |
| ------------------------ | -------------------------------------------- | --------------------------------------------- |
| **Desconto condicional** | Emissão automática ou sob demanda (não paga) | `descontoCondicionado`                        |
| **Desconto antecipado**  | Emissão pós-pagamento ou sob demanda (paga)  | `descontoIncondicionado`                      |
| **Juros/Multa**          | Apenas em pagamento após vencimento          | Soma ao valor final (não como campo separado) |

---

### Integridade

* O valor emitido da nota deve **coincidir com o valor financeiro registrado** (`account_receivables.paid_amount` ou `account_receivables.amount`).
* Nenhum campo deve resultar em soma indevida de descontos e acréscimos.
* A emissão deve considerar o **status da fatura**:
  `PAID`, `CANCELLED`, `PENDING`, `OVERDUE`, etc.

---

## 🧩 Alteração Técnica

### 1️⃣ Incluir campos necessários no `include` de `account_receivables`

Atualizar o método `emitManyNfse` para incluir os campos que influenciam o cálculo:

```diff
const receivables = await this.prismaService.account_receivables.findMany({
  where: { uuid: { in: ac_ids } },
  include: {
    business: { select: { uuid: true, business_no: true } },
    client: {
      select: {
        business_no: true,
        business_name: true,
        email: true,
        address_relations: true,
      },
    },
+   payments: { select: { amount: true, paid_at: true } },
+   discounts: true,
+   fines: true,
  },
});
```

---

### 2️⃣ Atualizar `buildEmitPayloadFromReceivable`

Adicionar cálculo dinâmico do valor emitido:

```ts
private calculateEffectiveValue(receivable: any): {
  valorServico: number;
  descontoCondicionado: number;
  descontoIncondicionado: number;
} {
  const status = receivable.status;
  const isPaid = status === 'PAID';
  const isOverdue = receivable.due_date && dayjs(receivable.due_date).isBefore(dayjs());
  const modalidade = receivable.emission_mode; // 'AUTOMATIC', 'AFTER_PAYMENT', 'ON_DEMAND'
  let valorServico = Number(receivable.amount);
  let descontoCondicionado = 0;
  let descontoIncondicionado = 0;

  // Valor pago (caso exista)
  const valorPago = receivable.paid_amount ?? 0;

  switch (modalidade) {
    case 'AUTOMATIC':
      valorServico = Number(receivable.amount);
      descontoCondicionado = Number(receivable.discount_conditional ?? 0);
      break;

    case 'AFTER_PAYMENT':
      valorServico = valorPago || Number(receivable.amount);
      descontoIncondicionado = Number(receivable.discount_anticipation ?? 0);
      break;

    case 'ON_DEMAND':
      if (isPaid) {
        valorServico = valorPago;
        descontoIncondicionado = Number(receivable.discount_anticipation ?? 0);
      } else {
        valorServico = Number(receivable.amount);
        descontoCondicionado = Number(receivable.discount_conditional ?? 0);
      }
      break;
  }

  // Juros/multa (apenas para atrasos)
  if (isOverdue && isPaid) {
    valorServico += Number(receivable.interest_amount ?? 0);
  }

  return { valorServico, descontoCondicionado, descontoIncondicionado };
}
```

---

### 3️⃣ Aplicar o cálculo no payload

Substituir o trecho atual do método `buildEmitPayloadFromReceivable`:

```diff
valor: {
-  servico: Number(receivable.amount),
-  descontoCondicionado: 0,
-  descontoIncondicionado: 0,
+  servico: effective.valorServico,
+  descontoCondicionado: effective.descontoCondicionado,
+  descontoIncondicionado: effective.descontoIncondicionado,
},
```

e incluir o cálculo logo antes:

```ts
const effective = this.calculateEffectiveValue(receivable);
```

---

### 4️⃣ Logs e rastreabilidade

Adicionar logs detalhados para rastrear o cálculo:

```ts
this.logger.log(
  `[NF Emissão] Fatura ${receivable.uuid}: modo=${receivable.emission_mode}, ` +
  `status=${receivable.status}, valor_emitido=${effective.valorServico.toFixed(2)}`
);
```

---

## 🧾 Exemplo de Comportamento Esperado

### Caso 1 — Fatura automática (não paga)

```
Fatura: R$ 1.000,00
Desconto condicional: R$ 50,00
Emitida automaticamente → Valor da nota = R$ 950,00
```

### Caso 2 — Fatura paga após vencimento

```
Fatura: R$ 1.000,00
Pagamento com multa e juros: R$ 1.050,00
Emitida após pagamento → Valor da nota = R$ 1.050,00
```

### Caso 3 — Emissão sob demanda (paga)

```
Fatura: R$ 1.000,00
Valor pago: R$ 980,00
Emitida manualmente após pagamento → Valor da nota = R$ 980,00
```

---

## 🧪 Testes e Validação

| Cenário                             | Valor Esperado                       |
| ----------------------------------- | ------------------------------------ |
| Fatura pendente, emissão automática | = valor bruto - desconto condicional |
| Fatura paga antes do vencimento     | = valor pago - desconto antecipado   |
| Fatura paga após vencimento         | = valor pago + juros/multa           |
| Emissão sob demanda (não paga)      | = valor bruto - desconto condicional |
| Emissão sob demanda (paga)          | = valor pago - desconto antecipado   |

---

## ⚠️ Cuidados Técnicos

* Validar se `receivable.paid_amount` está populado (em alguns ambientes pode vir `null`).
* Caso a modalidade não esteja configurada, assumir **modo automático** como padrão.
* Garantir que o valor emitido nunca seja negativo (aplicar `Math.max(valor, 0)`).

---

## ✅ Resultado Esperado

* Emissões fiscais refletem com precisão o valor contábil.
* Descontos e juros são aplicados conforme regras de negócio.
* Nenhuma divergência entre nota fiscal e financeiro.

---

**Status:** Pronto para desenvolvimento
**Data:** 07/11/2025
**Versão:** `v1.0.0`

```


```
