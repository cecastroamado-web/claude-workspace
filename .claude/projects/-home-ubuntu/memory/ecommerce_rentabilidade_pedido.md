---
name: Rentabilidade por pedido — modelo e rateio
description: Como /api/profitability calcula CMV, ICMS/DIFAL, frete e comissões — incluindo rateio por invoice_id e shipment_id, e fix dos bugs de NaN
type: project
originSessionId: 59d02a22-8a24-4195-af56-bf66adc8af06
---
# Rentabilidade por pedido — ecommerce-agent (mai/2026)

## Pipeline /api/profitability (api.py:4211)

Carrega:
- **ML orders paid** com rateio de ICMS/DIFAL e frete proporcional ao `total_amount` quando múltiplos pedidos compartilham `invoice_id` (mesma NF-e ML) ou `shipment_id` (mesmo envio). Antes era 50/50 → pedidos pequenos ficavam com imposto/frete inflado.
- **Bling NF-e** situacao=6, exclui EBAZAR.
- **ADS** rateado por TACOS (ad_total / ml_revenue).
- **NFS-e** SEPARADO em `/api/services-profitability` (consolidado no front quando empresa="todas").

## CMV — sem fallback estimate (mai/2026)

`metrics.py:compute_profitability` removeu o estimate enviesado. `cmv_source`:
- `real` — todos itens com custo cadastrado
- `parcial` — alguns itens sem custo
- `sem_custo` — nenhum item com custo
- `sem_itens` — NF-e sem itens sincronizados

Se cair em `parcial`/`sem_custo`/`sem_itens`, frontend mostra `~` (amarelo) ou `!` (vermelho) e job `check-nfe-sem-custo` (09h diário) alerta no Telegram.

## NaN — bugs históricos importantes

Pandas converte NULL → NaN. `cost_override is not None` retorna True para NaN. Fix em metrics.py: `cost_override is not None and not (isinstance(x, float) and pd.isna(x))` em **dois lugares** — CMV e crédito ICMS. Helper `_num()` no início do compute substitui `or 0` em campos numéricos do `order` (NaN é truthy → `or` deixava passar).

**Proteção global**: `SafeJSONResponse` em api.py sanitiza NaN/Inf → null em todas as respostas.

## Comissão ML × Comissão Vendedor (mai/2026)

Antes auto-rate de vendedor era somado em `comissao_ml` → poluía breakdown no canal externas. Fix:
- Coluna `comissao_vendedor` separada no profit_df
- Bling NF-e com `comissao` manual → vai pra `comissao_vendedor`
- Auto-aplicação em `api.py:4570` grava em `comissao_vendedor`
- Summary tem `total_comissao_vendedor`
- Frontend NÃO subtrai vendor de novo (backend já aplica em `lucro_liquido`) — antes tinha dupla subtração

## Modelo de custo Bling externo

Quando NF-e Bling externa não tem custo:
1. Sync items via `bling-nfe-items-sync` (job 6h) que chama `BlingClient.sync_nfe_items_to_db`
2. Custo resolvido por `_load_product_costs_with_map`: prod_costs por SKU/nome + nfe_product_cost_map (NF-e → produto interno) + product_kit_components
3. Casos pontuais: `cost_override` direto em order_items_detail (ex: HAVAN com substituição de componente — case 66 montado com imã 88 macho)

Exemplo HAVAN (NF 579, mai/2026): 740 case com imã 88 macho substituindo imã 66 padrão. cost_override = R$ 49,245 (corpo) + 4 × R$ 31,76 (imã 88 macho China) = **R$ 176,28/un**.

## Jobs no scheduler (mai/2026)

- `bling-nfe-items-sync` 6h — sincroniza items das NF-e Bling externas pendentes
- `check-nfe-sem-custo` 09h diário — alerta NF-e externas sem custo (cooldown via hash)
- `check-nfe-not-synced` mensal dia 1 09h30 — NF-e ML `invoice_status='not_found'`
- `weekly-summary` seg 08h — inclui bloco *Rentabilidade da semana* (helper `_period_profitability` + `_format_profitability_block`)
- `monthly-summary` dia 1 09h — bloco *Rentabilidade do mês fechado* + comparativo MoM

## Frontend — modal de detalhe (mai/2026)

`selectedOrder.impostos` virou objeto `{ federal, icmsLiq, difal, creditoIcms }`. Modal mostra breakdown em 3 linhas (Imposto Federal, ICMS Líq + crédito, DIFAL). Aba Comissões passa só `federal` com total agregado (modo degradado).

## Sinais de erro recorrentes a verificar primeiro

- ROI/margem zerados ou pedidos com `lucro=0` → buscar NaN no profit_df (provavelmente cost_override NULL)
- "Comissão ML" aparece em canal externas → comissao_vendedor sendo somada em comissao_ml de novo
- Imposto/frete super-estimado em pedido pequeno → query sem rateio proporcional por invoice_id/shipment_id
- "produtos sem vínculo" — alerta `/api/nfe-pending-costs` lista items sem cobertura
