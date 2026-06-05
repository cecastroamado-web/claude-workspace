---
name: ecommerce-cashflow-financiado
description: "Defaults e modos das simulações de financiamento no Fluxo de Caixa do ecommerce-agent (parcelamento impostos, Sicredi, capital de giro)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 2aa573e7-179b-452a-b981-85ca74a6452d
---

Página **Fluxo de Caixa · Simulações de financiamento** (ecommerce-agent) — `dashboard-ui/src/features/cashflow/index.tsx` + `agent/api.py` (`/api/cashflow`, `/api/provisao`). Ajustes de jun/2026:

**Parcelamento de Impostos** — defaults: prazo **12 meses** (limite que o estado libera atualmente), taxa **1,15% a.m.** (Selic), mês de início **01**. Frontend `financeConfig` + params backend `finance_taxes_months=12`, `finance_taxes_rate_am=0.0115`.

**Aplicação Sicredi** (alavancagem 1,5x e saldo bancário) — spread **0,40% a.m.** (`spreadAm: 0.0040`), Selic 1,15%.

**Empréstimo / Capital de Giro** — REFATORADO jun/2026 para **modo único** (commit 06ff74b). Removidos os modos `pmt` (banco propõe) e `comparar` (lado a lado) e o input de parcela do banco. Agora informa PV + nº parcelas + garantia + Selic + spread e escolhe o **sistema de amortização** (param `emprestimo_giro_sistema`):
- `price`: parcela fixa (Tabela Price).
- `sac`: amortização constante (pv/n) → parcela decrescente.
- `_simulate_emprestimo_giro(pv, n, rate_am, sistema='price', garantia=0, garantia_selic_am=0)` em api.py: taxa = Selic+spread sempre; gera `schedule` por mês pelo sistema. Retorna `sistema`, `pmt` (1ª parcela), `pmt_primeira`, `pmt_ultima`, `amortizacao_constante` (só SAC), `schedule`. `_distribute_emprestimo_giro_to_meses_proj` usa o schedule mês a mês (SAC entra no fluxo como série decrescente, não valor único). `_solve_price_rate_am` ficou órfão (inofensivo).
- Validado (PV 900k, 60x, 1,55% a.m.): Price = R$ 23.148,86 fixa, juros 488.931; SAC = R$ 28.950→15.232,50, amort 15.000 const, juros 425.475.
- **Garantia rendendo Selic** (params `emprestimo_giro_garantia` + `emprestimo_giro_garantia_selic_am`, default Selic 1,15%): lastro aplicado rende Selic (simples, sobre o principal travado pelo prazo) e abate o custo. Retorna bloco `garantia` com `rendimento_mensal/total`, `custo_liquido_total` (= juros − rendimento), `custo_liquido_mensal_medio`, `taxa_liquida_am` (≈ rate − (garantia/pv)·selic). Frontend envia garantia_selic_am = mesma Selic do config.
- **Caso real**: garantia 600k @1,15%/mês rende R$ 6.900/mês (R$ 414k em 60m) → custo líquido do giro cai de R$ 483k (juros) p/ **R$ 69k** (R$ 1.150/mês, taxa líquida **0,77%/mês**, abaixo da Selic). É a mesma alavancagem 1,5x da "Aplicação Sicredi".

**Proposta real Sicredi (jun/2026)**: garantia de **R$ 600k** → capital de giro de **R$ 900k** (alavancagem 1,5x), 60x de R$ 23.050. PV = **900.000** (não 600k!). Taxa efetiva **1,5333% a.m. (20,03% a.a.)** = Selic 1,15% + spread implícito **0,38%** — ótimo negócio, levemente abaixo do alvo Selic+0,40% (que daria R$ 23.148,86/mês). Banco é R$ 98,86/mês mais barato. Default UI: PV 900.000, 60x, sistema `price`.

**Detalhamento por tipo no Parcelamento de Impostos** (commit 06ff74b): cada adesão (`finance_taxes.adesoes[]`) agora traz `breakdown` {pis,cofins,icms,difal,das,irpjcsll}; a tabela de adesões expande a linha mostrando a composição. Esclarece por que certas competências são mais altas: (a) meses de fechamento trimestral (jul/out/jan/abr) incluem **IRPJ/CSLL**; (b) a competência do **"próximo mês"** usa base de receita que inclui as **externas/B2B do mês corrente** (ex.: NF Havan), inflando PIS/COFINS/ICMS — os meses futuros usam a projeção (menor). `_estimate_taxes_for(yr,mo)` em api.py é a fonte.

**Card "Aplicado em Renda Fixa" na Visão Geral** (commit 4ba7cc8): `/api/overview` retorna `saldo_rf_atual` + `saldo_rf_por_empresa` = saldo acumulado (aplicações − resgates efetivados, **histórico completo**, ignora filtro de período) das linhas `INVESTIMENTO RENDA FIXA` da planilha (empresa via `_is_aviation(CONTA)`). Card emerald em `overview/index.tsx` (componente `Overview`, NÃO `LoansSection` — esse usa LoansData). Validado: XConnect R$ 300.000, Aviation R$ 0.

Obs.: "SPV/SPE" que o usuário mencionou = empréstimo de **parcela fixa** (= Tabela Price), confirmado. Não é sistema de amortização exótico.
