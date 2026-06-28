---
name: ecommerce-cmv-mapeamento-kits
description: "Como mapear CMV de kits e itens sem custo no ecommerce-agent (product_kit_components, nfe_product_cost_map, cost_override) + decisões jun/2026"
metadata: 
  node_type: memory
  type: project
  originSessionId: 85b1fdaa-2bbf-46c3-bef8-79b101b760fb
---

Mapeamento de custo (CMV) de produtos no ecommerce-agent. Como o engine resolve o custo de um item de venda, por ordem de precedência (em `compute_profitability` / `_load_product_costs_with_map`):
1. **`order_item_component_override`** (por pedido/sku/item_id) — composição atípica de UMA venda.
2. **`order_items_detail.cost_override`** — custo fixo naquele item da venda (ganha do kit/lookup; ver caso NF478 abaixo).
3. **`nfe_product_cost_map`** (alias `nfe_product_name`→algo): se `kit_name` set → soma componentes do **`product_kit_components`**; senão se `cost_override` set → usa ele; senão `product_name`→`product_costs`.
4. **`product_costs`** por sku → item_id → product_name (lower).

**Kits = `product_kit_components`** (kit_name → linhas product_name/quantity/cost_override) + alias em `nfe_product_cost_map.kit_name`. **É DATE-AWARE**: `_load_product_costs_history_dict` chama `_load_product_costs_with_map(as_of_date=d)` por data, e o custo de cada componente é resolvido na data da venda. Componente com `cost_override` usa valor fixo; sem override busca `product_costs[nome]` vigente na data. ⚠️ Validar custo SEMPRE na data da venda (ex.: `_load_product_costs_with_map(db,'XConnect',as_of_date='2025-07-16')`) — na data de hoje dá outro valor. NÃO usar custo atual p/ venda antiga (erro contábil).

**Decisões gravadas (jun/2026, tudo é dado lido ao vivo — sem restart):**
- **Kit "Starlink Mini Gen 4 com Cabo 12v e Suporte Antena"** (sku `34`, XConnect, vendido via **Bling** B2B ~R$7.434/un, ex.: NF 478 Agropecuaria Maggi, NF 480 AMAGGI, 16/07/2025): `product_kit_components` = Antena (cost_override **799**) + **Super Case 88mm XConnect (Macho)** + **Cabo 12v 5m**. Date-aware: CMV/un **1.206,25** em 2025 (imã 250,20) → **1.083,09** em 2026+ (imã 127,04). Removido um `cost_override` manual (1.160,47) que existia no item da NF 478 p/ ficar consistente com a 480.
- **Aviation: "Suporte Case Injetado Branco/Preto/Cinza com Imas 88mm"** (NF 097/098/099) → alias `nfe_product_cost_map`→ **"Case Injetado 88mm Aviation"** (R$176,29 vigente). Mesmo padrão dos aliases de cor do 66mm.
- **XConnect: "Adaptador Carregador Veicular Carro Starlink Mini 118w Usbc"** (item ML `MLB5346979920`, sku `MLB5346979920_187492518119`, vendido via **ML** ~R$349, 3 vendas abr/2025): é um **carregador veicular WOTOBUS 118W** (USB-C 100W + USB-A 18W), NÃO cabo. Alias com **`cost_override=100`** (nome + sku) — não amarrado ao produto "Conector 12v - 118w" (que tinha R$200, alto demais). Margem subiu 7,5%→38,4%. Anúncio hoje pausado a R$499.

**Auditoria de CMV zerado (varredura 2 empresas, todos períodos via `_compute_profit_df_for_period(co,None,None,hoje)` filtrando `cmv_source in ('sem_custo','parcial')`):** após os 3 mapeamentos acima, **zero** itens sem custo (tudo `cmv_source='real'`). Repetir essa varredura periodicamente p/ achar novos produtos sem mapeamento.

Caveats: `product_costs` tem linhas duplicadas (UNIQUE não aplicada) mas o engine deduplica por componente (MAX effective_from). `cost_override` em `nfe_product_cost_map` NÃO é date-aware (valor único). Ver [[ecommerce-ventosa-utimex-credito-simples]] (Slim Pro 2) e [[ecommerce-bling-rules]].
