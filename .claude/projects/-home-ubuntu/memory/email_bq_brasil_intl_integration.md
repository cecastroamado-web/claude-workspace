---
name: BQ Brasil × Intl — integração já parcial, projeção é o gap real
description: get_unified_billing já junta Brasil+LATAM em BRL com schema idêntico; o que falta é alimentar CashFlowProjection com Brasil
type: project
originSessionId: c738a5e8-9708-47a0-9874-623b8cc12be8
---
A frase do MEMORY antigo "BigQuery Brasil — integração pendente" é parcialmente desatualizada. O estado real (10/mai/2026):

**Já integrado (Brasil + LATAM em mesmo schema):**
- `get_unified_billing()` em `bq_billing_reader.py:540` — 26 campos comuns, 0 exclusivos.
- Usado em: Executive Overview (KPIs MRR), Global — Ranking, FastAPI `/api/global-billing-by-pais`.

**Onde Brasil ainda é excluído ou tratado em separado:**
- Resultado por País — `app.py:1693-1696` exclui Brasil de `recv_prev_by_country`.
- Fluxo de Caixa Diário — Brasil usa `fetch_entradas_brasil` (BQ vencimento); Intl usa `fetch_entradas_from_projection`.
- Projeção / Cenários Runway / P&L por País — todos usam apenas Internacional.

**Why:** divergência arquitetural — Brasil vem do BQ tabela `faturamento` + planilha Despesas (cash basis), sem extrato bancário em `BANCO_TABS`. Internacional usa extratos por país + `payment_terms.json` (3 clientes hoje). Para Brasil entrar na `CashFlowProjection`, precisa: (a) adapter BQ→InvoiceRecord, (b) payment_terms para top clientes Brasil, (c) alocação wire_midia por país.

**How to apply:** ao planejar trabalho de integração, oferecer 3 caminhos:
- (A) Brasil lado a lado em rankings/P&L existentes — 1-2 dias, sem projeção
- (B) Projeção Brasil simplificada com prazo padrão (ex: 30d do vencimento) — 2-3 dias
- (C) Paridade com Internacional via payment_terms cadastrados — 1-2 semanas + decisões

Doc: `docs/diagnostico_bq_brasil_integracao.md`.
