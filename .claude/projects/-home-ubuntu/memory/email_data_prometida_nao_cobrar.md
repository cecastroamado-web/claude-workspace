---
name: email-data-prometida-nao-cobrar
description: "Convenção \"Não Cobrar\" no campo Data Prometida (Cobranza) — sinaliza ao time financeiro que cliente não aceita cobrança do time; CFO faz a cobrança pessoalmente quando atrasa"
metadata: 
  node_type: memory
  type: project
  originSessionId: 0d42e0cf-f4ec-4094-b3f8-6dfe2fd5b376
---

No campo **Data Prometida** das planilhas de Cobranza (Internacional), o valor literal `"Não Cobrar"` é uma convenção operacional, NÃO um sinal de exclusão.

**Why:** Alguns clientes (ex.: Renault no México via Grupo Satélite) não gostam de ser cobrados pelo time operacional. Nesses casos a fatura é legítima e deve ser paga, mas o time NÃO faz cobrança ativa — quem cobra é o CFO (Carlos Eduardo Castro Amado) pessoalmente quando entra em atraso.

**How to apply:**
- Faturas com `data_prometida == "Não Cobrar"` continuam sendo recebíveis válidos: aparecem no fluxo de caixa e em "A Recuperar" quando vencem. NÃO excluir nem marcar como anulada/neutra.
- Comportamento atual do código (`projection.py:122-136`): `_parse_date("Não Cobrar")` retorna `None`, cai no ramo de histórico → `expected = vcto + avg_delay`. Daí a fatura tem ciclo normal (entra no fluxo até vencer, depois vai pra atrasados). **Não mexer nessa lógica.**
- Se algum dia for útil destacar essas faturas no dashboard (ex.: tag "CFO collection"), basta detectar `data_prometida` ∈ {"Não Cobrar", "nao cobrar"} e propagar uma flag — mas NÃO usar isso para esconder ou excluir do fluxo.
- Exemplo real (mai/2026): faturas 9928/9929 do Grupo Satélite | Renault (Mexico), USD 1.758 cada, vcto 01/mai, com "Não Cobrar". O CFO vai cobrar diretamente quando entrarem em atraso significativo.

Relacionado: [[email-wire-intercompany]] (outra distinção semântica em recebíveis Internacional).
