---
name: ecommerce-sicredi-emprestimo
description: "ecommerce-agent — empréstimo Sicredi (capital de giro) na provisão: parcelas SAC com Selic mês a mês do BCB; RF em garantia fica bloqueada"
metadata: 
  node_type: memory
  type: project
  originSessionId: 826e85a9-889a-4588-8c15-645fd21d1225
---

# Empréstimo Sicredi (capital de giro) na provisão de caixa (10/jun/2026)

Empréstimo Sicredi contratado 10/jun/2026: **XConnect, R$ 450.000, 60 parcelas, SAC,
0,40% a.m. + Selic, 1ª parcela 25/07 (45 dias), demais a cada 30 dias.** A liberação já
caiu no Inter (entra no saldo de partida da provisão) → modelamos **só as parcelas** (saídas).

**Reserva de Renda Fixa (~R$300k): está EM GARANTIA do Sicredi → bloqueada.** NÃO entra
como caixa de partida na provisão (decisão CFO). O card "Aplicado em Renda Fixa" mostra
o valor, mas ele não cobre o déficit projetado.

## Selic mês a mês (Banco Central)
- `agent/bcb.py` `fetch_selic_mensal(months=12)` — API SGS do BCB, **série 4390** (Selic
  acumulada no mês, % a.m.). ⚠️ `/ultimos/24` dá HTTP 400; usar **12** (12 já basta).
  Devolve % (1.07) → converte p/ fração (0.0107). Mês corrente vem PARCIAL.
- Tabela `selic_mensal` (competencia 'YYYY-MM' PK, taxa_am fração). Job `selic-sync`
  (scheduler, startup +120s e 1×/dia) faz upsert. `GET /api/selic`.
- Provisão: taxa da parcela = Selic do **mês do vencimento** se conhecido e FECHADO
  (`competencia < mês corrente`), senão a **última Selic mensal fechada** (fallback
  `selic_fallback` da config). spread somado por cima.

## Config + projeção
- Tabela singleton `sicredi_loan_config` (enabled, empresa, principal, n_parcelas,
  spread_am, sistema 'sac'/'price', primeira_parcela, intervalo_dias, selic_fallback).
  Métodos `get/set_sicredi_loan_config` no db.py. `GET/POST /api/sicredi-loan`.
- Provisão (api.py, antes do "Para compatibilidade com o resumo"): se enabled e empresa
  bate, gera parcelas. **SAC**: amort const = principal/n; juros = saldo_devedor ×
  (selic_mês + spread); parcela = amort + juros (decrescente). **Price**: PMT fixo.
  Lança "Parcela Sicredi k/N (XConnect) (est.)" no vencimento. Só as que caem no
  horizonte (provisão tem **teto de 60 dias** → hoje só a 1ª, 25/07, aparece).
- Validado: 1ª parcela = R$ 14.115 (amort 7.500 + juros 6.615 @ 1,47% = Selic maio
  1,07% + 0,40%).
- ⚠️ **Carência de 45 dias da 1ª parcela é aproximada como 1 mês de juros** (não 1,5×).
  Refinar se o CFO confirmar a convenção do banco.

## Frontend
Card `SicrediLoanCard.tsx` na página Fluxo de Caixa (acima do ProvisaoCard): toggle
"considerar no fluxo", campos editáveis + preview (taxa atual = Selic BCB último mês
fechado + spread; 1ª parcela). Hooks `useSicrediLoan`/`useSaveSicrediLoan` (invalida
`provisao`). Tipos/métodos em `lib/api.ts`.

## Reserva de Renda Fixa (garantia) — rendimento a 96% do CDI (10/jun/2026)
A RF que cobre a garantia do Sicredi = aplicação de **R$ 300k em 02/06/2026, conta
"SICREDI X CONNECT"** (na planilha "INVESTIMENTO RENDA FIXA"). Rende **96% do CDI**.
- **CDI do BCB**: `agent/bcb.py fetch_cdi_mensal` (série **4391**, CDI acumulado no mês).
  Tabela `cdi_mensal`; o job `selic-sync` busca Selic (4390) **e** CDI (4391).
- **Acúmulo** (`api.py _cdi_accrual_factor`): fator = Π meses (1 + 0,96×cdi_mês); mês de
  início e mês corrente rateados por dias; **mês corrente vem PARCIAL do BCB → o
  rendimento atualiza sozinho** a cada carga. Meses sem dado → último fechado.
- **/api/overview** acumula SÓ as aplicações da conta **SICREDI** (a em garantia), desde
  a DATA da planilha. ⚠️ **NÃO somar todas as linhas RF**: há **161 linhas** de churn
  antigo na conta INTER (2024-25) que já zeraram (resgatadas) — somar tudo dava R$ 240k
  de rendimento FALSO. Filtro `"SICREDI" in CONTA` isola a aplicação real.
- Campos novos no overview: `rendimento_rf`, `saldo_rf_bruto` (=aplicado+rendimento),
  `rf_pct_cdi`, `rf_data_aplicacao` (auto da planilha). `GET/POST /api/rf-config`
  (pct_cdi override; data é auto). Card "Aplicado em Renda Fixa" mostra Aplicado /
  Rendimento (96% CDI) / Bruto + "em garantia, bloqueada".
- Validado 10/jun: 300k desde 02/06 = rendimento R$ 737 (8 dias), bruto R$ 300.737.

Ver [[ecommerce_provisao_pontos_cegos]], [[ecommerce_cashflow_financiado]] (simulação de
alavancagem Sicredi no /api/cashflow, separada disto).
