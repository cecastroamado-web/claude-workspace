---
name: ecommerce-havan-faturadas-antecipacao
description: Antecipação das NFs Havan JÁ FATURADAS pela data de entrega (seletor per-OC no fluxo de caixa)
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Antecipação de NFs Havan FATURADAS (jun/2026)

Quando o CFO fatura uma OC Havan, ela **sai do snapshot de OCs abertas** (`havan_pedidos_snapshot`) e vira um **recebível A RECEBER no Sheets** (FAVORECIDO HAVAN, venc ≈ entrega+121d). Por isso sumia do seletor per-OC "Quais OCs antecipar" (que só lista OCs abertas). Pedido do CFO: trazer as faturadas no MESMO painel, com **data de entrega editável** — as NFs são emitidas alguns dias ANTES da entrega (p/ a Havan agendar a programação).

## Como funciona
- Recebíveis faturados = Sheets A RECEBER + FAVORECIDO HAVAN. Helper `_havan_faturadas_receivables(rows, empresa, overrides)` em `api.py`.
- **Entrega por NF**: default = `vencimento − 121d` (`_HAVAN_FATURADA_PRAZO_DEFAULT`); CFO sobrescreve por NF. Persistido na tabela **`havan_faturada_entrega(nf, entrega, updated_at)`** (db.py). Endpoints `GET /api/havan/faturadas-detalhe` e `POST /api/havan/faturada-entrega`.
- **Antecipação**: recebe o LÍQUIDO no dia da ENTREGA (`eff_dt = max(entrega, hoje)`), descontado de `eff_dt → vencimento` pela taxa/modelo do havanConfig. Igual à mecânica das OCs abertas, mas a origem é o Sheets.
- **Reflete nos DOIS**: `/api/cashflow` (mensal, `meses_proj`) E `/api/provisao` (dia-a-dia, `a_receber_por_dia`). Campos de resposta `antecipa_havan_faturada`. Params: `antecipa_havan_faturada` (bool) + `antecipa_havan_faturada_nfs` (NFs vírgula; vazio=todas).
- **Sem dupla contagem** com o genérico `antecipa_havan` (que antecipa TODAS as A RECEBER Havan p/ HOJE): o bloco genérico **pula** NFs tratadas pelo seletor de faturadas (`_faturada_handled` / `_faturada_handled_prov`).

## UI (cashflow/index.tsx)
- Vive no mesmo toggle `anteciparAberto` das OCs abertas. Subseção "NFs já faturadas" com checkbox + `<input type="date">` por NF (salva via `setHavanFaturadaEntrega` → invalida query). Seleção em `antFatSel`.

## Distinção conceitual (importante)
- **OCs abertas** (`antecipa_havan_aberto`): ainda não entregues → caixa cai na entrega FUTURA.
- **Faturadas** (`antecipa_havan_faturada`): NF emitida dias antes da entrega → caixa cai na entrega (≈ hoje se entrega já passou). A "entrega prevista" só é editável aqui porque o Sheets só guarda o vencimento.

Relacionado: [[ecommerce-bling-rules]] (detecção de cancelamento + Sheets é fonte de caixa), [[ecommerce-havan-backlog-faturamento-transito]], [[ecommerce-cashflow-financiado]].
