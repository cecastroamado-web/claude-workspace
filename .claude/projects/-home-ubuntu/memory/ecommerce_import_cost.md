---
name: ecommerce-import-cost
description: Calculadora paramétrica de custo de importação (DI/nacionalização) e correção do ICMS no fluxo de imãs
metadata: 
  node_type: memory
  type: project
  originSessionId: 1b2e109b-851f-4399-8096-3490b1ba6d4e
---

Módulo `agent/import_cost.py` (ecommerce-agent) — recria/estende a planilha do despachante (ref. ATIVO AGX-26079) com a lógica tributária correta. Módulo de cálculo puro + CLI (Rich) + gerador `.xlsx` com fórmulas reais.

**Lógica (Brasil, Lucro Presumido comercial default):**
- VA = Mercadoria + Frete + Seguro (câmbio único).
- II = ii%×VA; **IPI = ipi%×(VA+II)** (base inclui II); PIS/COFINS = %×VA.
- ICMS-import TTD-SC: base por dentro `(VA+II+IPI+PIS+COFINS+despesas)/(1−destaque)`. **Destacado 4%** = crédito do cliente (pass-through, NÃO é custo). **Recolhido ~1,2%** = único desembolso real. **Spread 2,8 p.p.** = ganho.
- Regime: LP → só ICMS recuperável (II/IPI/PIS/COFINS viram custo, IPI não credita p/ comercial). Lucro Real → IPI/PIS/COFINS também creditam (parâmetro `regime`).
- Duas visões: **desembolso** (caixa) vs **custo efetivo** (desembolso − créditos).

**Validação vs planilha AGX-26079:** II/PIS/COFINS batem; **IPI da planilha subcalculado em R$ 7.728** (base ≈ CIF sem o II) → em LP cai direto no custo; ICMS zerado na planilha (recolhido real R$ 6.994). Câmbio da planilha era inconsistente por linha (merc 5,2275/frete 5,244/cabeçalho 5,23) — modelo usa câmbio único.

**Endpoints (api.py):** `POST /api/import-cost` (itens NCM USD/BRL, câmbio, despesas, ICMS, regime, cenários) e `GET /api/import-cost/agx` (referência + validação).

**Correção no fluxo de imãs (jun/2026):** o `_calc_nacionalizacao` antigo contava ICMS 4% cheio como custo (~R$ 21,8k) → nacionalização do pedido de imãs superestimada em ~R$ 14k. `/api/cashflow` agora calcula via `import_cost` (ICMS recolhido 1,2% = R$ 6,5k; destaque 4% é pass-through). Nacionalização do pedido caiu de ~R$ 182k → ~R$ 168k. Campos novos no response: `ima_desembolso`, `ima_custo_efetivo`, `ima_icms_destacado/recolhido/spread`, `ima_icms_recolhido_pct` (0.012 constante; config tem só `icms_pct`=destaque). `ima_nac_breakdown` ganhou o split TTD.

**Card do Fluxo (cashflow/index.tsx):** banner azul "Pedido de imãs realizado" REMOVIDO. Ficou só o card amber "Pedido de Imãs Importados" com alerta de recompra (estoque atual sem incoming) + "registrar pedido" colapsável; ao preencher mostra a simulação (federais, ICMS destacado/recolhido, spread, desembolso vs custo efetivo). Ver [ecommerce-cashflow-financiado](ecommerce_cashflow_financiado.md).

Relacionado: [ecommerce-icms-model](ecommerce_icms_model.md), [ecommerce-icms-aliquotas-reais](ecommerce_icms_aliquotas_reais.md).
