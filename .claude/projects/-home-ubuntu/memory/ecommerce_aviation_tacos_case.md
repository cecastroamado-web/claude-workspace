---
name: ecommerce-aviation-tacos-case
description: "Página /simulacao-tacos-aviation — TACOS/margem do case Aviation no ML; o gargalo é ADS, não preço"
metadata: 
  node_type: memory
  type: project
  originSessionId: 549997de-4398-4691-8053-e4718cd07ed6
---

Case Starlink Mini 66mm da **Aviation** (Simples Nacional) é o carro-chefe no ML (~260 ped/mês,
7 variações de cor). A margem real (engine `/api/profitability`) é **~22%**, espremida pelo
**ADS rateado (TACOS)**: subiu de ~10% (mar/2026) para **~19,6% (mai–jun/2026) = R$ 68–72/un** —
o maior custo controlável depois do CMV (~R$ 114–120/un). Diagnóstico-chave: **o problema é ADS,
não preço**. Concorrentes vendem semelhante a **R$ 319** (nosso preço médio ~R$ 352–368).

Análise (jun/2026): baixar p/ R$ 319 **sem cortar TACOS** derruba margem p/ ~16,4%; manter a margem
de hoje a R$ 319 exige **TACOS ≤ ~15,5%**; cortar TACOS p/ 13% rende mais que o corte de preço
(≈R$ 6 mil/mês no canal). Math exata: comissão/ADS/imposto são % da receita (escalam com o preço),
CMV e frete são fixos por unidade. TACOS de equilíbrio: `t* = 1 − com_rate − imp_rate − margem_alvo
− (frete+cmv)/preço`.

**Implementado (jun/2026):** página dedicada **`/simulacao-tacos-aviation`** ("TACOS / Margem ML
(Aviation)" no menu de Simulação), dados **ao vivo** da engine. 4 cards: diagnóstico de custo real,
ADS por campanha (ACOS/ROAS), **matriz Preço × TACOS** interativa (com TACOS de equilíbrio por preço),
e conclusão automática. Gera **PDF completo**.
- Backend: `GET/POST /api/aviation/case-tacos[/pdf]` (api.py) — reusa `_compute_profit_df_for_period
  ("Aviation")`, casa pedido ml_direct → MLB via `ml_orders_sync.item_ids` → família `_AVIATION_CASE_MLBS`.
- PDF: `build_ml_tacos_pdf()` em `havan_report.py` (recalcula a matriz no servidor; capa própria sem
  co-branding Havan).
- Front: `features/simulacao-tacos-aviation/` + rota + `sidebar-data.ts` + tipos/métodos em `lib/api.ts`.

Ver [[ecommerce-rentabilidade-padronizacao]] e [[ecommerce-comissoes-aviation-rate]]. Próximo passo
sugerido (não feito): alerta de TACOS no scheduler quando passar de um teto por produto/empresa.
