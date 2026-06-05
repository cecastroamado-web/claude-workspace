---
name: customer-ranking refatorado para usar compute_profitability
description: Em mai/2026 /api/customer-ranking foi refatorado para reutilizar /api/profitability + /api/services-profitability — antes superestimava lucro B2B
type: project
originSessionId: c55d1021-f78f-437e-a9da-511f2b2ea0d7
---
`/api/customer-ranking` foi refatorado em mai/2026 para chamar `get_profitability()` + `get_services_profitability()` internamente e agregar por (customer_name, company), em vez de replicar SQL próprio.

**Why:** A versão antiga somava manualmente receita - cmv - fees - imposto - desconto, mas omitia **comissão de vendedor** (campo `comissao` do bling_nfe) e **custo de frete** (`custo_frete`). Para Havan, o lucro reportado era R$ 152.453,72 quando o real é R$ 109.706,87 — diferença de R$ 42.747 (~ 28% inflado). Afetava todos os clientes B2B com vendedor ou frete.

**How to apply:** Quando alguém reportar divergência entre "Rentabilidade por Pedido" e "Rentabilidade por Cliente", verificar primeiro se /api/customer-ranking ainda está reutilizando os outros endpoints — não voltar a duplicar SQL. Validação: para Havan XConnect, ranking deve casar exatamente com soma de `lucro_liquido` dos pedidos no /api/profitability.

**Reconciliação validada:** Soma global do ranking = soma profitability (apenas com_cmv) + soma services-profitability. Clientes sem CMV cadastrado retornam `lucro=null` (intencional, para não exibir lucro fake).
