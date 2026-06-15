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

## Custo da operação — parcela × rendimento da garantia (12/jun/2026, commit `fa3f959`)
Pedido CFO: ver a evolução da parcela mensal lado a lado com o rendimento da RF em garantia
para apurar o **custo geral da operação**. Implementado:
- **`GET /api/sicredi-loan/schedule`** (api.py): cronograma COMPLETO das N parcelas (SAC/Price,
  juros = saldo × (Selic do mês + spread)) + rendimento mês a mês da RF em garantia (principal
  da conta SICREDI rendendo %CDI). Custo líquido do mês = juros − rendimento RF; **custo geral =
  juros totais − rendimento total da garantia** (a RF bloqueada abate o custo do crédito). CDI/
  Selic futuros = último mês fechado do BCB (taxas constantes — é projeção, não realizado).
- Helper **`_rf_garantia_state(empresa_norm)`** (api.py, perto de `_sicredi_loan_schedule`):
  reusa a fonte do card RF do overview (linhas INVESTIMENTO RENDA FIXA com CONTA contendo
  SICREDI; filtra churn antigo da INTER). Devolve principal, %CDI, cdi_map, last_cdi, rendimento
  até hoje.
- Frontend: `SicrediLoanCard.tsx` ganhou seção expansível "Custo da operação" — 3 KPIs (juros,
  rendimento da garantia, custo líquido) + tabela parcela a parcela + rodapé de totais. Hook
  inline `api.sicrediLoanSchedule(empresa)`; tipos `SicrediSchedule`/`SicrediScheduleRow` em
  api.ts.
- **Validado** vs DB real: PV 450k/60x SAC @1,47% → 1ª parcela R$14.115 (bate c/ provisão); juros
  totais R$201.758 (44,8% PV); garantia 300k @96%CDI (composto) rende **R$253.886** no prazo.
- **Custo NOMINAL composto = −R$52.128 era ENGANOSO** (CFO questionou, com razão). Decompondo: o
  empréstimo AMORTIZA (SAC, saldo médio R$228.750, cai abaixo dos 300k já na parcela #22) enquanto
  a garantia CRESCE (saldo médio R$411.938, 1,8× a exposição) e COMPÕE; e a soma é nominal
  (ignora valor do dinheiro no tempo). Os 3 fatores juntos produzem o negativo contraintuitivo.
- **Modelo final (commit `6b78d37`, decisão CFO): garantia COMPÕE (juros compostos = realidade da
  RF) + apresenta o custo em VALOR PRESENTE.** Juros são front-loaded, rendimento composto é
  back-loaded; trazer ambos a hoje é a comparação justa. **Taxa de desconto = SELIC** (commit
  `c7b0fa7`, decisão CFO): NÃO a taxa do empréstimo — o spread de 0,4% é prêmio de risco de crédito
  do banco, não custo de oportunidade puro; ambos os fluxos são de baixo risco → desconta-se pela
  taxa básica livre de risco. **PV juros R$163.740 − PV rendimento R$180.672 = PV custo líquido
  −R$16.932 (≈ neutro, −3,8% do PV).** (Selic 1,07% e a própria RF 96%CDI 1,03% dão ~o mesmo
  ~−17k, o que reforça a escolha; com a taxa do empréstimo 1,47% daria −8.264.) Card mostra custo
  nominal + destaca o custo em VP com a explicação. Campos novos:
  `desconto_am`, `pv_juros_total`, `pv_rendimento_total`, `pv_custo_liquido_total`,
  `pv_custo_efetivo_pct`, `rf_saldo_final` (composto ≈ R$553.886). (Houve um passo intermediário
  com RF simples = custo +R$16.861, descartado: RF real compõe.)
- **MODELO FINAL (commit `b3c26bb`, decisão CFO 13/jun): CUSTO EFETIVO de crédito com garantia.**
  O "custo líquido" (juros − rendimento) dava negativo enganoso (−16.932) e o CFO recusou ("o banco
  jamais perde"). Insight do CFO: a garantia de R$300k rende ~Selic (96% CDI, confirmado que rende
  mesmo penhorada — bloqueada só p/ saque), então na parte do saldo COBERTA pela garantia você paga
  só o **spread líquido** (taxa empréstimo − rendimento garantia ≈ 0,44%/mês = spread 0,40% + gap
  Selic−96%CDI 0,04%); só o **EXCEDENTE** (saldo acima de R$300k, R$150k inicial) paga a **taxa
  cheia** Selic+spread. No SAC o excedente encolhe e zera quando o saldo fica todo coberto (parcela
  #22). **Custo efetivo: R$76.953 nominal / R$64.306 VP (≈17,1% do liberado, ~3,4%/ano)** vs juros
  cheios R$201.758/R$163.740 — a garantia abate ~R$124.805. O banco sempre ganha ≥ spread (positivo).
  Fórmula/parcela: `coberto×max(0,taxa−rend_garantia) + excedente×taxa`. Campos: `coberto`,
  `excedente(_inicial)`, `net_rate_coberto_am`, `custo_efetivo_total`, `pv_custo_efetivo_total`.
  Card mostra decomposição coberto/excedente por parcela. (Histórico descartado: "custo líquido"
  nominal/VP e a fase de RF simples.)
- ⚠️ Tudo é projeção a **taxas constantes** (último mês fechado do BCB) por 60 meses — extrapolação
  longa; sinalizado na UI. Refinar com curva futura de Selic/CDI se o CFO quiser.

## IOF — auto-detectado do Sheets (15/jun/2026)
O IOF é cobrado de uma vez logo após a liberação (já debitado no caixa → NÃO entra nas parcelas
futuras da provisão; só no CUSTO da operação). Lido AUTOMATICAMENTE da planilha p/ cobrir operações
futuras sem reconfig: helper `_sicredi_iof_auto(empresa_norm)` (api.py, perto de `_rf_garantia_state`)
soma os débitos cuja DESCRIÇÃO DA CONTA='IOF' na conta **SICREDI** da empresa (efetivados, valor<0).
Override manual opcional: coluna **`iof`** em `sicredi_loan_config` (NULL=auto). `/api/sicredi-loan`
devolve `iof`(override) + `iof_auto`; `/schedule` soma o IOF ao custo: `custo_total_com_iof` (nominal)
e `pv_custo_total_com_iof` (IOF em t=0 → PV=nominal) + `_pct`. Card `SicrediLoanCard.tsx` tem campo
IOF editável (vazio=auto) e a seção "Custo da operação" virou **Custo total da operação (c/ IOF)** =
crédito + IOF. **Validado 15/jun: IOF real R$14.000,16 (12.290,16 diário + 1.710,00 adicional=0,38%×450k),
liberado 10/06/2026; custo crédito R$76.952,70 + IOF → custo total R$90.952,86 nominal (20,2% do PV) /
R$78.305,72 VP.** Migração `ALTER TABLE sicredi_loan_config ADD COLUMN iof REAL` no db.py.

Ver [[ecommerce_provisao_pontos_cegos]], [[ecommerce_cashflow_financiado]] (simulação de
alavancagem Sicredi no /api/cashflow, separada disto).
