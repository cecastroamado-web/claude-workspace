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

Ver [[ecommerce_provisao_pontos_cegos]], [[ecommerce_cashflow_financiado]] (simulação de
alavancagem Sicredi no /api/cashflow, separada disto).
