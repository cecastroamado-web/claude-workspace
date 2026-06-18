---
name: ecommerce-havan-oc-55267-recancel
description: "PENDENTE investigar — OC Havan 2026-55267-1 com NF cancelada p/ refaturar não voltou a \"a faturar\" e itens seguem \"em trânsito\""
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# ✅ RESOLVIDO (18/jun): OC Havan 2026-55267-1 — NF cancelada p/ refaturar

**Causa raiz + fix (18/jun):** `_havan_em_transito_por_sku` (api.py) lia `havan_nfe_items` filtrando **só por data de emissão** — não excluía NF cancelada. A 587 (slim 150 + haste 400) ficava presa "em trânsito" e a OC não voltava pra "a faturar". **Fix:** JOIN com `bling_nfe` + filtro `situacao=6 AND COALESCE(exclude_from_revenue,0)=0` (a detecção de cancelamento já marca exclude_from_revenue=1). Resultado: slim/haste da 587 saíram do em-trânsito; OC 2026-55267-1 (R$ 50.750) já tinha voltado ao snapshot de pedidos abertos (varredura 06h) → "a faturar" mostra os 150 slim + 400 haste de novo. Commit no submódulo.

---

## (histórico) PENDENTE: OC Havan 2026-55267-1 — NF cancelada p/ refaturar (17/jun/2026)

O CFO **cancelou a nota** da OC **2026-55267-1** (Havan) para **refaturar**. Sintomas (bug de consistência):
- A OC **NÃO voltou a aparecer como "a faturar"** no painel (deveria reabrir, já que a NF foi cancelada).
- Os itens que estavam faturados nela — **400 hastes + 150 Slim Pro** — **seguem aparecendo como "em trânsito"** (faturado-não-recebido) no painel da Havan.

Causa provável (mesma família do fix de cancelamento de NF-e, ver [[ecommerce-bling-rules]]): quando uma NF Havan é cancelada, o "em trânsito" (`_havan_em_transito_por_sku` / `havan_nfe_items`) e o "a faturar" (`pedidos_abertos − em_transito_qtd`) não recalculam — a NF cancelada continua contando como faturada/em-trânsito, e o saldo da OC não volta pra "a faturar".

A investigar:
- `havan_nfe_items` (sync lazy do Bling) — a NF cancelada some da lista `situacao=6` mas o item antigo fica gravado → em-trânsito fantasma. Mesma lógica do detector de cancelamento que fiz no `_run_bling_nfe_db_sync` (re-checa situacao + `exclude_from_revenue`), mas aqui é na tabela `havan_nfe_items` / no cálculo de em-trânsito e a-faturar.
- Ver [[ecommerce-havan-dashboard]] (conciliação, em-trânsito por SKU, a-faturar) e [[ecommerce-havan-portal-pedidos]] (OCs).

Não investigado/corrigido ainda — só anotado a pedido do CFO em 17/jun.
