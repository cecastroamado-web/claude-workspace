---
name: email-projecao-baixa-incompleta
description: "Sintoma \"Projeção EOM internacional subiu após pagamento\" = baixa incompleta (WIRE In no banco sem Data Pagamento na aba Receitas)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 71c9cf23-0ba9-499b-980a-646218421c4c
---

Quando a Projeção Final do Mês na página "Projeção" (tab Internacional) **sobe** depois que um cliente paga, o motivo quase sempre é **baixa incompleta**: o WIRE In foi lançado no extrato do banco mas a aba Receitas do país ainda não tem a coluna "Data Pagamento" preenchida.

**Why:** Fórmula em `agent/financeiro/projection.py:746` — `projected_eom = saldo + recv_month − remaining_expenses − remaining_wire_out`. O filtro de fatura paga (linha 123-125) só remove do `recv_month` se `(cliente.lower(), ano_ref, mes_ref)` estiver em `paid_periods`, que vem de Data Pagamento na aba Receitas. Quando saldo sobe mas recv_month não cai, a projeção infla pelo valor recebido.

**How to apply:** Se o usuário relatar "a projeção saltou sem motivo", primeiro pedir para conferir nas linhas da aba Receitas dos pagamentos recentes:
1. Data Pagamento preenchida
2. Cliente na linha == nome usado na fatura/Bling (matching é fuzzy mas precisa ser próximo)
3. Competência (mes_ref/ano_ref) da fatura == mês/ano da linha

Caso confirmado que está tudo certo na planilha mas o problema persiste, aí investigar cache do Streamlit em `load_payment_history` e o reader que constrói `paid_periods`. Relacionado: [[email-projection-filtros]].
