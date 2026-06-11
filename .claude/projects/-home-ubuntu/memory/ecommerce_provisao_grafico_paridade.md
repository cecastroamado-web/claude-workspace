---
name: ecommerce-provisao-grafico-paridade
description: "ecommerce-agent — feedback do CFO: tudo que entra na provisão de caixa precisa aparecer também no gráfico do fluxo (paridade provisão × /api/cashflow)"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 826e85a9-889a-4588-8c15-645fd21d1225
---

**Princípio (CFO, 11/jun/2026): tudo que é contabilizado na PROVISÃO de caixa
(`/api/provisao`, tabela diária do ProvisaoCard) precisa aparecer TAMBÉM no GRÁFICO
do fluxo (`/api/cashflow`, projeção mensal `projecao_real`) — senão a visão do fluxo
fica incorreta.**

**Why:** são dois endpoints/visões diferentes da mesma realidade. A provisão é diária
(teto 60 dias); o gráfico é mensal (12-13 meses). Se um item (parcela de empréstimo,
imposto, ADS, etc.) é adicionado só na provisão, ele some do gráfico e o saldo
projetado mensal fica errado. O CFO olha o GRÁFICO pra decisão.

**How to apply:** ao adicionar qualquer saída/entrada nova na provisão
(`get_provisao`, em `saidas_est_por_dia` / releases / a_receber), adicionar a MESMA
coisa no `/api/cashflow`:
- provisão diária: `saidas_est_por_dia[data].append({...})`.
- gráfico mensal: bucketizar no `meses_proj[(ano, mês)]` numa chave própria, **subtrair
  no `saldo_periodo_base`** e **expor no dict de `projecao_real`**; no frontend
  (`cashflow/index.tsx` `monthLines`) adicionar o rótulo na lista de débitos/créditos.
- ⚠️ o dict interno do `meses_proj` tem chaves fixas (factory) — usar
  `mp[k] = mp.get(k, 0.0) + v` (NÃO `mp[k] += v`, dá KeyError em chave nova).
- Sempre que possível, extrair a lógica num **helper compartilhado** usado pelos dois
  (ex.: `_sicredi_loan_schedule` serve provisão e cashflow → não divergem).

Caso que originou: a parcela do empréstimo Sicredi estava só na provisão e não
aparecia no gráfico em 07/2026. Ver [[ecommerce-sicredi-emprestimo]].
