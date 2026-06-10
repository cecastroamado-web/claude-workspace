---
name: ecommerce-provisao-pontos-cegos
description: "ecommerce-agent — provisão de caixa: fix dupla contagem (ML creditado hoje) + pending vencido + saldo MP disponível (jun/2026)"
metadata: 
  node_type: memory
  type: project
  originSessionId: b988fa55-62a8-429d-bc3c-83c51c398212
---

# Provisão de Caixa — dupla contagem + pontos cegos (jun/2026)

`/api/provisao` tinha 1 bug de dupla contagem e 2 pontos cegos. Modelo: saldo
parte do banco (Inter, planilha `STATUS=OK` via `get_bank_balance`) + entradas ML
por `money_release_date` (`get_pending_releases_real`, status `pending`). Payouts
(saques MP→Inter) NÃO entram na provisão — só em `/api/ml-payouts` p/ conciliação.

## Fix (a) — dupla contagem do ML já creditado no banco hoje
`ml_ja_creditado_hoje` (planilha STATUS=OK, DATA=hoje, "venda") já está no
`saldo_inicial`. Era calculado e devolvido na resposta mas **nunca subtraído** →
venda liberada+sacada+lançada no mesmo dia contava 2×. Fix em `api.py` bloco 4.1:
abate `min(ml_ja_creditado_hoje, net_real_hoje)` do bucket de hoje (clamp).

## Fix (b1) — pending VENCIDO descartado
`get_pending_releases_real` filtrava `release_date >= hoje`; pending com data
programada já vencida (ML ainda não liberou) sumia. Fix: chamar com `date_from =
hoje−60d` e bucketizar `rel < hoje` em HOJE. Validado: XConnect hoje-próprio
R$660,68 + vencido R$3.032,86 = R$3.693,54; Aviation R$1.536,15 + R$5.111,69 =
R$6.647,84 (bate com endpoint).

## Fix (b2) — saldo disponível no MP (já liberado, não sacado)
Dinheiro liberado pelo ML mas não transferido pro Inter ficava em nenhuma ponta.
Lido do **BALANCE_AMOUNT final do Release Report** (última linha-DADO; footer sem
DATE zera o campo — validado: bate com NET_CREDIT do footer). Pequeno pois o CFO
saca com frequência: XConnect R$548,91 / Aviation R$1.860,41 (02/06).
- `db.py`: tabela `ml_account_balance` (company PK, balance, as_of) +
  `upsert_account_balance`/`get_account_balances`.
- `mercadolivre.py`: `fetch_mp_balance(begin,end)` (mesmo download do `fetch_payouts`).
- `scheduler.py`: job `payouts-sync` grava o saldo (passo c).
- `api.py`: bloco 4.2 lê `get_account_balances`, soma ao saldo de partida; resposta
  expõe `saldo_mp_disponivel`, `saldo_mp_por_empresa`, `saldo_atual_total`.
- Front: `ProvisaoCard.tsx` tile "Saldo atual" mostra total + banco + "MP a sacar";
  tipos em `lib/api.ts`.

## Fix (c) — released de HOJE descartado (09/jun/2026)
A entrada ML usava só `get_pending_releases_real` (status `pending`). Pedidos que
viram `released` durante o dia eram excluídos (premissa: já sacados pro banco).
Mas o saque MP→Inter ocorre **parcial ao longo do dia / só à noite** → released-de-
hoje ainda não está no banco e sumia (`saldo_mp_disponivel` foi zerado em
[[ecommerce-provisao-release-staleness]]). Sintoma: "a liberar hoje" combinado
542,84 vs realidade ~3.013 (XC 0,0 / AV 596,78). Causa: AV = pending 1.486,78 −
creditado 890 = 596,78; XC pending c/ retenção zerado pelo creditado 1.000.
- **Fix:** novo `db.get_released_today_real(company, day)` (status `released` AND
  `release_date == hoje` — só HOJE; released de dias anteriores seguem fora, preserva
  o fix de 07/jun contra dupla contagem com o banco). Bloco 4 (api.py) tem loop (a.2)
  que bucketiza os released-de-hoje em HOJE com a mesma matemática (retenção 25% XC).
  O abate 4.1 (`ml_ja_creditado_hoje`, mesma fonte planilha que `get_bank_balance`)
  remove a parte já sacada → seguro mesmo com saque parcial intradiário.
- Pós-fix hoje: XC entrada projetada 624,91 / AV 2.388,23 / combinado 3.013,14.

## Linha "A liberar (painel ML)" (09/jun/2026)
Campo por dia `entradas_ml_bruto` (`slot["bruto_real"]`) + total
`total_a_liberar_bruto_periodo`. ProvisaoCard mostra 2 linhas: **"A liberar"**
(= painel ML "a liberar") e **"Entrada projetada"** (= alimenta o saldo, líq. de
retenção/abate). `entradas_ml_real`/`total_releases_reais_periodo` deixaram de ser
"= painel ML" (são a entrada líquida projetada). Tipos em `lib/api.ts`.

**DEFINIÇÃO CORRETA (validada ao vivo 09/jun com saque do CFO):** "a liberar" do
painel = **só PENDING** (retido, ainda não liberado), líquido de taxas **E da
retenção 25%** (o ML já desconta o empréstimo antes de exibir o "a liberar" do
XConnect). Acumula **`entrada` (= net − retenção), NÃO `net`**, e SÓ no loop dos
pending — **released-de-hoje NÃO entra** (já virou "disponível"/sacável no painel; some
quando o CFO saca). Validação 1:1: AV 597,15 e XC 226,45 = painel.
- **Erro inicial (corrigido na mesma sessão):** 1ª versão somava pending+released e
  usava `net` (sem retenção) → bruto superestimava o painel após qualquer
  liberação/saque (AV mostrava 3.278 vs painel 597; XC 322,89 vs 226,45). O teste ao
  vivo (saque MP→Inter das 2 empresas) expôs isso: released-de-hoje continua `released`
  no nosso DB mesmo após sacado, então não pode entrar no "a liberar".
- O **saldo/projeção não é afetado** por essa linha (é só display); os released-de-hoje
  seguem na entrada projetada via fix (c) acima, com o abate evitando dupla contagem.
- ⚠️ **`ml_ja_creditado_hoje` vem em valores REDONDOS manuais** (1.000/2.400/890/3.490)
  porque o **reconcile do Inter está quebrado (certs mTLS faltando)** — sem reconcile
  automático o abate fica aproximado e cada saque lançado redondo desalinha um pouco a
  projeção. Priorizar baixar os certs do Inter (`certs/xconnect.crt`,`certs/aviation.crt`).

## Vencimentos reais de impostos + ADS por empresa (10/jun/2026)
Datas de vencimento na provisão (`api.py`, in-loop com `_tax_month`=mês seguinte +
gap-filler do mês fechado), **todos mês subsequente à apuração**:
- **PIS/COFINS** XConnect → dia **25** · **ICMS** → dia **12** (antes eram 1 lump no
  dia 10; split preserva o total e residualiza cada tipo separado) · **DIFAL** → dia
  15 (inalterado) · **DAS** Aviation → dia **20** · **IRPJ/CSLL** → dia 30 trimestral.
- ICMS XConnect = ICMS ML estimado (proxy mês anterior, residual) **+** ICMS Bling real
  das NF-e (canais distintos, somam). ⚠️ o gap-filler do mês fechado NÃO inclui o Bling
  real (pré-existente; só o in-loop inclui) — possível melhoria.

**ADS — janela móvel 10d + datas por empresa:**
- Sync (`scheduler._run_ml_ads_sync`) busca gasto dos últimos 10 dias COMPLETOS por
  campanha (`MercadoLivreClient.get_campaign_metrics_range`) → tabela `ml_ad_spend_rolling`
  (company PK, cost, days). `ml_ad_spend` segue mensal (sem diário).
- Provisão: bloco dedicado por empresa (`_ads_by_co` = rolling_daily/mtd_real/last_actual;
  `_ADS_PAY_DAY = {XConnect:5, Aviation:25}`). **Vencimento = dia 05 (XC) / 25 (AV) do mês
  SEGUINTE ao gasto** (arrears). Mês de gasto: corrente = real-até-hoje + média_10d ×
  dias restantes; mês fechado = gasto REAL (vence neste mês, gap-filler); futuros =
  média_10d × dias. Substituiu o "mês corrente ÷ dia" que inflava com promo no início
  (Aviation "Liquida 6.6" → 31k). Validado: AV maio real 15.260 vence 25/jun; XC junho
  proj. ~4.98k vence 05/jul.

Testes `tests/` 307/307 OK (`test_pending_releases.py` 8/8). Ver [[ecommerce_ads_fix]], [[ecommerce_money_release_date]],
[[ecommerce_mp_payouts]], [[ecommerce_provisao_model]], [[ecommerce-provisao-release-staleness]].
