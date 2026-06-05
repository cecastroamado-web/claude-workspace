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

Testes `tests/test_pending_releases.py` seguem 8/8 OK. Ver [[ecommerce_money_release_date]],
[[ecommerce_mp_payouts]], [[ecommerce_provisao_model]].
