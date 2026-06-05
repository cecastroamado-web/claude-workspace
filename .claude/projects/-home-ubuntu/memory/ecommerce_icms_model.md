---
name: Cálculo ICMS Líquido — ecommerce-agent
description: Como ICMS, DIFAL e impostos são calculados na projeção de caixa e provisão
type: project
originSessionId: c649a731-4d80-4827-aab5-55bec76396a7
---
## FONTE ÚNICA (jun/2026): ICMS deriva de `compute_profitability`

Desde jun/2026 `_calc_icms_net_month` **delega para `_icms_net_month_from_profitability`** (api.py), que roda a MESMA lógica da rentabilidade (`compute_profitability`, metrics.py — crédito por item, taxa date-aware, override de componente, ML+B2B) via helper `_compute_profit_df_for_period`, somando `icms_liq`/`credito_icms`/`difal` por mês. Fallback ao cálculo antigo (abaixo) só em exceção.
- **Motivo:** 2 implementações divergentes; gráfico/provisão/DRE mostravam ICMS ≠ da rentabilidade (maio: 28.914 vs 27.546). Agora TODAS batem (maio XConnect: líquido 27.546, crédito 15.749, DIFAL 10.478).
- **Telas unificadas (todas via _compute_profit_df_for_period → compute_profitability):** rentabilidade, gráfico cashflow, provisão diária, DRE, **/api/order-detail** (branch NF-e, XConnect) e **/api/commission-report**. order-detail/commission-report puxam credito_icms/icms_liq do profit_df do mês (match por bling_order_id), fallback ao ad-hoc fora de escopo. Ex.: order-detail HAVAN NF 579 (com override de componente) corrigido — estava super-creditando.
- **Dead code removido:** `_effective_tax_rate` (sem callers). **Não unificados (de propósito):** `eff_tax_rate`/`_tax_by_co` (taxa % só display, não afeta caixa) e IM88 (dormente, _IM88_MONTHLY_CASES=0).
- **Crédito B2B (HAVAN etc.):** a rentabilidade já creditava insumos de vendas Bling via `nfe_product_cost_map`; o cálculo antigo NÃO (débito B2B entrava cheio → inflava). Mapeamento do cabo HAVAN (#HVN#) adicionado em `nfe_product_cost_map`. `product_costs` tem linhas duplicadas no mesmo effective_from → subconsulta de crédito deve dedup por id (senão multiplica).
- Provisão/gráfico **residualizam o DIFAL e impostos**: estimativa cheia − parcial já lançado na planilha por competência (override). Ex.: DIFAL maio 10.478 − 2.630 parcial = 7.847. Provisão monta o mapa de overrides igual ao gráfico (campo COMPETENCIA, fallback vencimento−1mês).
- DRE anual chama 12×/empresa mas roda ~2s.

## Função `_calc_icms_net_month(company, ym, conn)` — agent/api.py — FALLBACK (cálculo antigo)

Calcula ICMS líquido mensal para projeção fiscal. Aviation retorna zeros (ICMS incluso no DAS).

**ICMS débito** = `SUM(ml_nfe.valor_icms)` + `SUM(bling_nfe.valor_icms)` do mês
**ICMS crédito** = `qty × custo × icms_credit_rate` via `order_items_detail` × `product_costs`
- Alíquotas: 17% (nacional), 4% (importado), 0% (exceção)
- Itens B2B sem match em product_costs = crédito 0 (conservador)
**ICMS líquido** = max(débito − crédito, 0)
**DIFAL** = `SUM(ml_nfe.valor_icms_ufdest + valor_fcp_ufdest)` — Aviation = 0 (Simples)

**Why:** Flat 13,8% estava errado — NF-e real varia 7-12% por estado. Crédito de insumos reduz ICMS a pagar. Resultado março XConnect: débito R$53.457, crédito R$5.949, líquido R$47.508, DIFAL R$27.786.

## Modelo de impostos por empresa

### XConnect (Lucro Presumido)
- PIS: 0,65% faturamento → dia 10 mês seguinte
- COFINS: 3,00% faturamento → dia 10 mês seguinte
- ICMS: `_calc_icms_net_month().liquido` → dia 10 mês seguinte
- DIFAL: `_calc_icms_net_month().difal` → dia 15 mês seguinte (ML coleta via GNRE)
- IRPJ: 8% × 15% = 1,20% faturamento trimestral → dia 30 mês pós-fechamento (4,7,10,1)
- CSLL: 12% × 9% = 1,08% faturamento trimestral → dia 30 mês pós-fechamento

### Aviation (Simples Nacional)
- DAS: 11% faturamento → dia 20 mês seguinte
- DIFAL: R$0 — ML não coleta DIFAL para Simples Nacional
- bling_nfe.valor_icms = 0 (ICMS incluso no DAS, não destacado em NF-e)

## Onde é usado

- `/api/cashflow` projecao_real: `impostos_est` por mês (PIS+COFINS + ICMS + DIFAL + DAS + IRPJ/CSLL)
- `/api/provisao`: lump sums dia 10 (PIS/COFINS/ICMS), dia 15 (DIFAL), dia 20 (DAS), dia 30 (IRPJ/CSLL)
- `scheduler._run_monthly_tax_launch`: lança na planilha A PAGAR dia 1 de cada mês

## Base de faturamento (mês anterior real)
- ML: `ml_orders_sync` WHERE `status='paid'` AND `strftime('%Y-%m', data)=ym`
- NF-e: `bling_nfe` WHERE `situacao=6` AND sem EBAZAR
- NFS-e: `bling_nfse` WHERE `situacao=1`
- NÃO usar `data_hora` (coluna não existe) — usar `data`
- NÃO usar `valor_servicos` em bling_nfse — usar `valor_nota`

## Bugs corrigidos (abr/2026)
- `data_hora` → `data` em ml_orders_sync (provisão estava quebrando silenciosamente)
- `valor_servicos` → `valor_nota` em bling_nfse (coluna não existe)
- EBAZAR filter adicionado à query de faturamento (bling_nfe)
- DIFAL XConnect estava ausente de projecao_real e provisão

## Bug CRÍTICO no crédito ICMS — corrigido mai/2026

**Sintoma**: crédito de ICMS quase zerava o débito (R$28.805 de crédito vs R$29.416 de débito), quando o correto é ~R$13.377.

**Causa raiz**: a query JOIN com `product_costs` não incluía `component_id` na cláusula GROUP BY/PARTITION. Isso fazia o SQLite somar TODOS os `cost_value` históricos (versão Brasil + versão China) de cada componente, em vez de pegar apenas o preço vigente mais recente.

**Fix definitivo**: usar subconsulta correlacionada com `MAX(effective_from)` por `(company, product_name, sku, component_id)`:
```sql
JOIN (
    SELECT pc1.company, pc1.product_name, pc1.sku, pc1.component_id,
           pc1.cost_value, pc1.icms_credit_rate
    FROM product_costs pc1
    WHERE pc1.effective_from = (
        SELECT MAX(pc2.effective_from) FROM product_costs pc2
        WHERE pc2.company = pc1.company AND pc2.product_name = pc1.product_name
          AND pc2.sku = pc1.sku AND pc2.component_id = pc1.component_id
          AND pc2.effective_from <= ?  -- ym_end = 'YYYY-MM-28'
    )
) pc ON ...
```

**Resultado validado abril 2026 XConnect**:
- Débito ML: R$29.416 | Bling: R$0 | Total: R$29.416
- Crédito: R$13.377 (Case 66mm corpo 17%=R$8.003 + imã 4%=R$2.483 + demais)
- Líquido: R$16.039
- DIFAL: R$21.106

**ATENÇÃO**: Sempre que editar a query de crédito ICMS em `_calc_icms_net_month()`, verificar que `component_id` está na subconsulta correlacionada. Sem isso, versões históricas de preço se somam e inflam o crédito.
