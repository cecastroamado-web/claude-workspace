---
name: projecao-ml-cashflow
description: Bases de projeção ML no /api/cashflow + regra de propagar a base atual para projetar impostos futuros (não usar mês anterior fixo)
metadata: 
  node_type: memory
  type: project
  originSessionId: 449ba2d4-8551-480d-80ba-72da7b475561
---

## Bases disponíveis (ml_proj_base)

Cinco opções em `agent/api.py:get_cashflow`:

| Opção | Cálculo | Característica |
|---|---|---|
| `ma_30d` (DEFAULT) | Soma dos últimos 30 dias corridos | Mais responsivo a inflexões; default desde mai/2026 |
| `weighted_3m` | 0.5×m-1 + 0.3×m-2 + 0.2×m-3 | Antigo default; suaviza com meses fechados, lag de inflexão |
| `prev_month` | Mês calendário anterior | Legado; ignora tendência |
| `ma_60d`, `ma_90d` | Soma N dias / N × 30 | Mais estável, menos responsivo |

Todas devolvem **receita LÍQUIDA** (`total_amount - sale_fee - shipping_cost`) — vai pro caixa direto.

## Mês corrente (`ml_cur_estimate`)

Desde mai/2026: **extrapolação linear pura** (`cur_partial × dias_no_mes / dia_atual`). Não faz mais blend com base — antes era `cur_linear × peso + base × (1-peso)`, o que inflava no início do mês quando base era otimista.

`_peso` continua exposto pra contexto mas não é mais usado no cálculo.

## Para impostos: use a base BRUTA da mesma janela

PIS, COFINS, DAS, IRPJ/CSLL incidem sobre faturamento **bruto** (não líquido). Bug histórico: para meses futuros usava `_pm_ml_xc` / `_pm_ml_av` (mês anterior fechado, **fixo**) — não acompanhava queda de venda. Fixes em mai/2026:

- Novo helper `_ml_revenue_gross_query(conn, company, conds)` — soma `total_amount` SEM deduzir fees/shipping.
- Variáveis `ml_monthly_avg_gross_xc` e `ml_monthly_avg_gross_av` calculadas pela **mesma janela do ml_proj_base ativo**.
- Três pontos corrigidos (todos no `agent/api.py:get_cashflow`):
  1. `_estimate_taxes_for` branch "else" (usado pela simulação de parcelamento)
  2. **Loop principal** que popula `impostos_est` em `projecao_real`, branch "else" (meses 2+)
  3. **Loop principal** branch `_is_next_mo` (próximo mês — antes usava abril fixo + externas reais, agora usa `ml_monthly_avg_gross_*` + externas reais)
- ICMS escala proporcionalmente em todos os 3: `_pm_icms_xc[k] × (rev_proj / _pm_ml_xc)`.

**Mês corrente NÃO usa essa base**: precisa do mês anterior fechado + externas parciais para semântica fiscal correta — ainda não há histórico do próprio mês de competência. O fix só vale para próximo mês + meses 2+ no futuro.

## ADS proporcional à receita projetada

Mesma família de bug: o ADS em `ads_est` para meses futuros usava `ads_monthly_last30` (valor fixo do mês anterior) — não acompanhava queda de venda projetada.

Fix em mai/2026:
- Calcula `_ads_revenue_ratio = _prev_ads_val / _prev_gross_val` (clipado entre 1% e 30% pra evitar outlier de mês com ADS zerado ou venda anômala).
- Meses futuros: `ads_est = (ml_monthly_avg_gross_xc + ml_monthly_avg_gross_av) × _ads_revenue_ratio`.
- Mês corrente continua extrapolação pro-rata real (`ads_monthly_current`).
- Em mai/2026 com ma_30d ativo: ADS jun/26+ caiu de R$ 35.919 → R$ 19.270 (mantém 9% sobre faturamento, como em abril).

## Como verificar se uma estimativa de imposto faz sentido

1. Olhar `tax_base_xc` e `tax_base_av` em cada item de `projecao_real` — é a receita usada como base.
2. Para meses futuros (>= mes+2), deve ser próximo de `ml_monthly_avg_gross_*` (não do mês anterior).
3. Próximo mês (_is_next_mo): base = `ml_monthly_avg_gross_*` + externas reais já emitidas (pode subir muito se houve NF Bling grande no mês corrente, ex.: Havan).
4. Mês corrente: base = mês anterior fechado + externas parciais (semântica fiscal).
5. Se o % imposto/receita_ml_est parecer absurdo (>50%), lembrar que `receita_ml_est` no mês corrente é "falta receber" (R$ líquido pendente), não faturamento total — não dá pra comparar.
6. Alíquotas: PIS 0,65%, COFINS 3%, ICMS varia, IRPJ/CSLL ~3,08% × presunção 8/12 (só em meses 4/7/10/1).
