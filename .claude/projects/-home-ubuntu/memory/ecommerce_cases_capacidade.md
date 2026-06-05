---
name: Cases — capacidade de produção por tipo de imã
description: Como calcular capacidade de cases a partir do estoque de imãs no ecommerce-agent (F e M são modelos separados, somam)
type: project
originSessionId: 90d12968-730f-47ba-962b-4efe86e42327
---
## Regra de cálculo

Cada modelo de case usa **um único tipo de imã** (não ambos):

- Case **fêmea** 66mm usa só `IM-66-F` (4 imãs/case)
- Case **macho** 66mm usa só `IM-66-M` (4 imãs/case)
- Idem para 88mm com `IM-88-F` / `IM-88-M`

Por isso F e M **somam** na capacidade total, não competem.

| Tipo | Cálculo | Exemplo (saldo mai/2026) |
|---|---|---|
| 66mm total | (saldo IM-66-F + IM-66-M) ÷ 4 | (231 + 199) / 4 ≈ 106 cases |
| 88mm total | (saldo IM-88-F + IM-88-M) ÷ 4 | (8836 + 6780) / 4 ≈ 3904 cases |

**Why:** Sócio confirmou em 11/mai/2026 após eu erradamente afirmar que a capacidade 88mm estava "limitada pelo macho" (assumindo que cada case precisava de 4F + 4M). Não é isso — F e M são SKUs de cases diferentes.

**How to apply:** Ao analisar o endpoint `/api/imas` ou `/api/cashflow`, somar os cases F+M do mesmo diâmetro para reportar capacidade total. Nunca tomar `min(F, M)` como gargalo — não é como o produto funciona. O JSON do `/api/imas` já devolve `simulacao[].cases` por SKU separadamente; é só somar pares F+M.

## Pipeline produzíveis no /api/cashflow (jun/2026)

`_PIPELINE_PRODUCIBLE = cases_66 + cases_88` (api.py ~3655). **Ventosa (VEN-PRO ÷3) NÃO entra** — é outro produto. Os imãs **88mm em estoque serão usados pra montar cases** (decisão do sócio jun/2026), por isso entram nos produzíveis. Custo do imã 88: **R$ 31,76/un** (preço médio ponderado; por enquanto só o pedido HAVAN usou, via cost_override — ver [[ecommerce_rentabilidade_pedido]]). Antes a pipeline somava ventosa (produzíveis 4.773 → 3.750 ao remover; runway 10,9 → 8,9 meses). Saldo real jun/2026: IM-66 ~430 imãs (106 cases, **66mm quase esgotado**), IM-88 ~14.576 (3.644 cases), VEN-PRO 3.069.

## Empréstimo ML — quitação com saldo real (provisão, jun/2026)

A provisão lê o **pago-até-hoje** da planilha (lançamentos `EMPRESTIMO` + favorecido `MERCADO LIVRE`, data ≥ início 10/04/2026 — filtro de data evita somar empréstimos ML anteriores). `saldo_devedor = _EMPRESTIMO_VALOR_TOTAL (246.027,99) − pago`; quitação projeta só o saldo restante a partir de hoje. Pagamentos reais (batem com painel ML): abr 61.901,48 + mai 36.225,20 (jun 1.120,03 lançado no fechamento do mês). Resposta expõe `valor_pago`/`saldo_devedor`. Cada pagamento = par venda+EMPRESTIMO net-zero na planilha (ver [[ecommerce_money_release_date]]).
