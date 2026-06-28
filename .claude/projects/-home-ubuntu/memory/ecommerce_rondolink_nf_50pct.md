---
name: ecommerce-rondolink-nf-50pct
description: RONDOLINK emite NF a 50% do real (reduzir frete) — receita 2x via customer_revenue_factor; excedente do Sheets = venda sem nota
metadata: 
  node_type: memory
  type: project
  originSessionId: 85b1fdaa-2bbf-46c3-bef8-79b101b760fb
---

**RONDOLINK SOLUCOES LTDA** (cliente B2B XConnect): a pedido do cliente, a NF-e é emitida por **50% do valor real** (pra reduzir o frete, calculado sobre o valor declarado). O cliente paga **100%** — o dinheiro entra na conta e é lançado no **Sheets** como "VENDA DE PRODUTOS Rondolink" (todas STATUS=OK; **nenhum recebível pendente**). As NF-e ficam `status_pagamento='pendente'` no Bling só por falta de reconciliação (não são recebíveis reais).

**Problema original:** painel mostrava margem NEGATIVA da Rondolink (receita = 50% da NF, mas CMV = custo cheio).

**Modelo de tratamento (decidido CFO, jun/2026), reconcilia com o caixa:**
```
Sheets recebido (caixa real) ........ R$ 44.923
├─ Com NF (5 notas × 2) ............. R$ 30.430  → CMV real (itens NF), imposto sobre a NF
└─ Diferença = 100% sem nota ........ R$ 14.493  → CMV estimado 45%
```
- **CMV% sem nota = 45%** (derivado do mix da própria Rondolink: CMV real R$13.767 ÷ receita real R$30.430).

**FASE 1 — IMPLEMENTADA (fator de receita):**
- Tabela **`customer_revenue_factor(company, customer_pattern, factor, active, note)`** (criada em api.py setup + seed RONDOLINK/XConnect=2.0). Loader `_load_customer_revenue_factors(company)`.
- Param **`customer_revenue_factors`** no `compute_profitability` (metrics.py): se `customer_name` casa o pattern, `lucro += receita×(fator−1)` e `receita_bruta = receita×fator`. **Excedente = margem PURA**: NÃO recalcula imposto (pago sobre a NF) nem CMV (já é o real). Passado nos 4 call sites (panel `_compute_profit_df_for_period` com company; alerts/ai/vendor-roi sem filtro). É **gerencial** — NÃO altera receita fiscal (bling_nfe).
- Resultado: 5 NFs Rondolink saíram de margem negativa p/ **44–54%**; receita painel R$30.430, lucro +R$15.204.

**FASE 2 — IMPLEMENTADA (Opção A — linha sintética):** o balde 100% sem nota (R$14.493, CMV 45%) entra como **linha sintética** no painel. Tabela **`customer_semnota_adjust(company, customer_pattern, year, revenue, cmv_pct)`** (seed RONDOLINK/XConnect, year=**0**=sentinela "todos os períodos", revenue=14493, cmv_pct=0.45) + loader `_load_semnota_adjusts(company, year)`. Em `_compute_profit_df_for_period`, após o compute_profitability, injeta 1 row: `_source='semnota'`, cmv_source='estimado_semnota', receita=revenue, cmv=revenue×cmv_pct, sem imposto/comissão, lucro=receita−cmv. Aparece só quando o ano pedido casa (year=0 → só na visão "todos os períodos", senão dobraria entre anos). **Resultado Rondolink consolidado = R$44.923 receita (= caixa do Sheets ✅), lucro R$23.175, margem 52%.**
- ⚠️ **NÃO usar year=NULL** na tabela (NULL != NULL quebra o UNIQUE → seed duplica → linha injetada N×). Usar **year=0**.
- **Separado por ANO** (snapshot jun/2026): linhas em `customer_semnota_adjust` → **year=0** (total R$14.493,29), **year=2025** (R$10.353,29), **year=2026** (R$4.140,00). Fórmula por ano: `sem-nota(ano) = Sheets recebido(ano) − 2×NF(ano)`. 2025: 28.663,29−18.310; 2026: 16.260−12.120. Loader mostra a linha do ano filtrado (year=0 só na visão "todos"). Sem dupla contagem.
- ⚠️ É **SNAPSHOT**: atualizar `revenue` (year=0 e por ano) quando o acumulado Rondolink mudar. Recalcular com o script (lê NFs do DB + recebido do Sheets por ano). NÃO é auto-derivado.
- NÃO foi pela auditoria `receita_sem_nota` (que é export ad-hoc, não feature integrada); manual_sales não alimenta o painel de rentabilidade (só receita/comissões).

Para futuras NFs Rondolink o fator 2× se aplica automático (match por customer_name). Ver [[ecommerce-cmv-mapeamento-kits]].
