---
name: runway-de-cases-tem-duas-metricas-distintas
description: "O cashflow do ecommerce-agent expõe runway \"ritmo atual\" e \"pior caso\" — não confundir e não usar uma para o propósito da outra"
metadata: 
  node_type: memory
  type: project
  originSessionId: 449ba2d4-8551-480d-80ba-72da7b475561
---

## Duas perguntas, dois cálculos opostos

O painel `/api/cashflow` responde DUAS perguntas diferentes sobre o estoque de cases. Confundir as duas gera resultado contra-intuitivo (ex.: vendas caindo + runway diminuindo).

| Pergunta | Campo no response | Base de venda mensal | Quando usar |
|---|---|---|---|
| Quanto tempo o estoque vai durar? (previsão realista) | `pipeline_months` / `pipeline_months_with_incoming` | `ml_monthly_units` = 30d | Display do "Ritmo atual" |
| Quando devo fazer o próximo pedido pra não furar? (proteção) | `pipeline_months_pico` / `pipeline_months_with_incoming_pico` | `ml_units_pico` = max(30d, 90d, trend) | Trigger de overdue, `ima_next_order_ym` |

**Why:** Em mai/2026 o user pegou um bug onde vendas caíram (1.221 → 408) mas o runway mostrava 5,1 meses (pior que antes). Eu tinha confundido as semânticas e usado `max()` como base única — pior caso é correto pra disparar pedido, mas faz o display ficar contra-intuitivo. Fix: separar em duas métricas, mostrar lado-a-lado.

**How to apply:**
- Card "Runway de cases" mostra os dois (Ritmo atual + Pior caso). Cor do alerta usa pior caso.
- `imaEffectiveOverdue` e `ima_next_order_ym` usam pior caso (antecipa reposição).
- Se for adicionar nova métrica de estoque, pergunte: "isto é previsão ou trigger?" → usa o campo correspondente.

## Tendência: regra de incluir mês corrente normalizado

A `ml_units_trend` (regressão linear sobre 4-6 meses anteriores) **inclui o mês corrente extrapolado** quando já tem ≥ 7 dias passados:

```python
qty_normalizada = qty_realizada × dias_no_mes / dias_passados
```

Sem isso a inflexão recente nunca entrava na regressão (mês incompleto era excluído) e a tendência ficava cega para quedas iniciadas no mês corrente.

**Why:** Antes, em maio/2026, a regressão usava só nov→abr e projetava 1.566 un/mês ignorando a queda de maio. Com normalização, projeta 1.232 — captura a inflexão.

**How to apply:** Se mexer no cálculo da tendência (`_units_trend_val` em `agent/api.py`), garanta que o mês corrente continua entrando normalizado. Pular ele de novo reabre o bug.
