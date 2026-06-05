---
name: ROI por vendedor (vendor-roi-report)
description: Lógica do endpoint /api/vendor-roi-report, filtros de competência e interpretação do ROI
type: project
originSessionId: 59d02a22-8a24-4195-af56-bf66adc8af06
---
# ROI por vendedor — ecommerce-agent (mai/2026)

Endpoint: `GET /api/vendor-roi-report?empresa=todas&year=&until_ym=YYYY-MM`

## Fórmula

```
custo_total = comissao_devida + custo_rh
resultado = lucro - custo_total
roi = resultado / (cmv + custo_rh)
```

- **comissao_devida**: comissão calculada pelas vendas do período (vendor_rates × lucro_op ou comissao manual da NF-e). Usa o que vai ser pago, não o já pago.
- **comissao_paga**: tracking — não entra no custo_total.
- **custo_rh**: despesas do Sheets cujo FAVORECIDO bate com o nome do vendedor (lista que existir em vend_map ou pago_by_vend), exceto descrição "COMISSOES" e "INSUMOS".

## until_ym (cap de competência) — fix mai/2026

Default `until_ym` = mês anterior ao atual. Limita:
- `commission_payment_records`: filtra `ciclo_mes <= until_ym`
- Custos RH do Sheets: filtra `(row_year, mes_competencia) <= (until_year, until_month)`

**Importante**: usa `_comp_year_month(r)` (parseia COMPETENCIA dd/mm/yyyy) — NÃO usa coluna `ANO` da planilha porque linhas de despesas futuras saem com `ANO=1899` (default Excel para data inválida) e escapam do filtro. Bug HAVAN (mai/2026): R$ 48.800 de despesas COMERCIAL futuras de Rafael Ruczinski escapavam pelo filtro de ANO.

## Comportamento esperado

- Em 05/2026, default `until_ym=2026-04` → considera vendas até hoje × despesas até abril.
- Para projeção de ROI futuro: passar `until_ym` mais à frente.
- Para ano fechado: passar `year=2025&until_ym=2025-12`.
