---
name: ecommerce_ima_sugestao_pedido
description: "Sugestão de quantidade de imãs no fluxo de caixa — demanda combinada ML+Havan, cobertura lead+6m (decisões do CFO jun/2026)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 04aa6148-ffed-4da9-bc7c-97d934b195f6
---

# Sugestão de pedido de imãs — demanda combinada ML + Havan (jun/2026)

Adicionado ao `/api/cashflow` (ecommerce-agent) + card de imãs no Fluxo de Caixa.

**Bug que originou:** alerta "Hora de pedir imãs" mostrava texto contraditório
("runway ~4,3m ≤ lead ~3,5m"). O gatilho real é `(_ima_next_order_dt - now).days <= 30`,
i.e. **runway combinado − lead ≤ ~1 mês**, não runway ≤ lead. Texto corrigido + novo
campo `ima_order_in_days`.

**Achado:** a velocidade de venda usava **só `ml_orders_sync`** (Mercado Livre). A Havan
compra o **mesmo case** ("SUPORTE CASE PLASTICO C/IMAS STARLINK XCONNECT", SKU
HVNCSPLIM66007V) do mesmo estoque de imãs, mas não entrava na conta → runway
superestimado.

**Erro intermediário (corrigido):** primeiro estimei Havan por `valor_NF ÷ R$295`, o que
inflou 4–10× (a NF mistura cabos/slim/haste/ventosa). O correto é o **produto case
específico do snapshot Havan** (`havan_snapshot.data_json`), não o valor da NF.

**Decisões do CFO (trava as fórmulas):**
- Base de demanda = **ML + Havan**.
- Sinal Havan = **sell-out** (`vendas_mensais` do snapshot, média móvel 3 meses completos,
  exclui mês corrente). ~63 cases/mês. NÃO sell-in (recebido é lumpy + Havan sobre-estocada).
- Cobertura alvo = **lead (3,5m) + 6 meses = 9,5m**.

**Implementação (`agent/api.py` get_cashflow):**
- `_havan_units` = sell-out MA do case no snapshot (só XConnect).
- `_demand_combined` / `_demand_combined_pico` = ML + Havan; runway dos imãs passou a usar combinado.
- Sugestão: `qtd = max(0, demanda_pico_combinada × 9,5 − _PIPELINE_CASES)`, arredonda
  ↑ lote 4.000 imãs (1.000 cases). Campos `ima_suggested_magnets/cases/coverage_months`,
  `ima_demand_ml/havan/combined`, `ima_pipeline_cases_now`, `ima_coverage_target_months`.
- Frontend: quebra de demanda + sugestão com botão "usar sugestão" (seta `imaForm.magnets`).

**Validação live (04/jun/2026):** ML pico 1.133 + Havan 63 = 1.196 cases/mês; pipeline
4.902 cases; runway 4,1m; pedido em ~17 dias; **sugestão 28.000 imãs (7.000 cases) →
cobertura 10m**.

Relacionado: [[ecommerce_havan_backlog_faturamento_transito]], [[ecommerce_import_cost]],
[[ecommerce_cashflow_financiado]].
