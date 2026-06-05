---
name: finance-agent-project
description: "Dashboard de finanças pessoais (PF) com Pluggy + Claude + Telegram + React. 5 fases completas em mai/2026. 57 arquivos, ~30 endpoints REST, 13 loops async, 13 tabelas SQLite."
metadata: 
  node_type: memory
  type: project
  originSessionId: 6bba168a-4b65-4526-8779-76b59916bf50
---

Sexto projeto do workspace: `/home/ubuntu/finance-agent/`. Dashboard de finanças pessoais integrado a Open Finance Brasil via Pluggy, com categorização AI Claude, alertas Telegram e UI React.

**Why:** usuário pediu sistema automatizado para controle financeiro pessoal em tempo real (receitas, despesas, projeção, anomalias). Construído mai/2026 em ~1 dia.

**How to apply:** ao retomar, ler `/home/ubuntu/finance-agent/PLAN.md` (status das 5 fases + endpoints + jobs detalhados) e `/home/ubuntu/finance-agent/README.md` (setup + troubleshooting). Stack espelha o `ecommerce-agent` (FastAPI + aiosqlite WAL + React/Vite/TS + systemd + scheduler async com loops `_loop_*` e `scheduler_state`).

## Estado final (2026-05-18, 5 fases completas)

**Backend:** 16 módulos Python, ~30 endpoints REST, 13 loops async, smoke test passa (deps instaladas, imports OK, DB init com 75 categorias + 10 rules seeded, FastAPI sobe com 29 paths `/api/*` + JWT).

**Frontend:** 7 rotas (login + 6 páginas auth), 13 components em `features/`. Não foi rodado `npm install`/`npm run build` aqui — falta isso em deploy.

## Resumo das fases entregues

### Fase 1 — Conexão + sync + dashboard mínimo
- `pluggy_client.py` REST async com retry 401/429
- `sync.py` com dedup pagto fatura cartão (regex `PAGTO|PAGAMENTO|PGTO + FATURA|CARTAO`)
- `webhook.py` :8513 com HMAC SHA256
- `scheduler.py` com full-sync 6h + health 1h + catchup + watchdog
- Páginas: Overview, Contas (Pluggy widget), Transações

### Fase 2 — Categorização Claude
- Taxonomia PF: 15 parents × ~5 children = 75 categorias seeded
- 10 regras-semente (Pix, Netflix, Uber, postos, etc.)
- `categorizer.py` com Claude Haiku batch 20 + prompt caching agressivo
- Tools API com `enum` apenas de IDs de folhas (não pais) — evita alucinação
- `promote_feedback_to_rules` cria regra após 3 correções iguais do mesmo merchant
- UI: CategoryCell inline, Treemap CSS-grid, RulesPanel

### Fase 3 — Recorrências + projeção
- `recurring.py` detecta mensal (28-32d), quinzenal (13-16d), weekly (6-8d)
- Critérios: ≥3 ocorrências, std interval ≤3d, CV valor ≤20%, sinal consistente
- `cashflow.py` projeção D+90 = saldo BANK + recorrentes futuras + fatura CC no due_day + média móvel 90d por DOW (excluindo merchants já recorrentes)
- UI: ProjectionChart SVG nativo (sem Recharts), zona vermelha sob threshold, marcadores em eventos discretos

### Fase 4 — Anomalias + insights AI
- 4 detectores: outlier (z-score ≥2.5, requer ≥10 samples na cat), duplicate (mesmo merchant+amount em <48h via self-join), overspend (MTD >1.5× média 3m, após dia 10), new_merchant_large (≥R$500)
- `dedup_key` UNIQUE evita duplicação entre runs
- High severity → Telegram agrupado com cooldown 1h
- `ai_insights.py` daily brief Sonnet 4.6 com prompt caching + cache 12h em `ai_cache`
- Snapshot rico: saldo, top 5 cats, delta vs média 3m, anomalias, projeção 30d, recorrências próximos 7d

### Fase 5 — Budgets
- `budgets.py` BudgetTracker com tone (ok/warn/caution/over em 60/80/100%)
- Rollover: soma sobra do mês anterior
- Categoria pai agrega filhas em `category_spend_for_budget` (LEFT JOIN parent)
- Alertas 80% e 100% via Telegram (1 vez cada por cat+ym)
- UI: BudgetGauge SVG semicircular, BudgetForm com optgroup pai/filha, BudgetOverviewCard no overview

## Pontos críticos do desenho

1. **Dedup pagamento de fatura CC** — Pluggy entrega lançamento na fatura E débito na CC. Flag `is_credit_card_payment` populada na ingestão; `/api/overview` exclui da soma; `/api/transactions` filtra por default.
2. **MFA Pluggy expira ~90d** — webhook + health-check 1h → alerta Telegram + badge UI; botão "Reconectar" chama `connect-token` com `item_id`.
3. **JWT_SECRET default** — `Config._jwt_secret()` falha hard; escape em dev com `FINANCE_ALLOW_DEFAULT_JWT=1`.
4. **Categorização só em folhas** — Claude recebe `enum` apenas com IDs de categorias-folha (não pais), evita alucinação fora da taxonomia.
5. **Categorias-pai em budgets** — `category_spend_for_budget` agrega filhas via LEFT JOIN parent.
6. **Time zone** — schedules em UTC com offset +3 (não DST-aware, OK pós-2019 sem horário de verão no Brasil).
7. **Prompt caching obrigatório** — categorização (taxonomia + 15 few-shots cacheados) e daily brief (instruções cacheadas) reduzem 90% do custo.

## Schemas (13 tabelas)

```
Fase 1: pluggy_items, accounts, transactions, scheduler_state
Fase 2: categories, category_rules, ai_feedback
Fase 3: recurring, cashflow_projection
Fase 4: anomalies, ai_cache
Fase 5: budgets
```

Todas com WAL + foreign keys + busy_timeout 15s.

## Endpoints (29 sob /api/*)

`/api/health`, `/api/pluggy/{connect-token,items}`, `/api/accounts`,
`/api/transactions` + `/recategorize-pending` + `/{id}/category`,
`/api/overview`, `/api/categories[/{id}|/treemap]`, `/api/rules[/{id}]`,
`/api/recurring[/{id}|/detect]`, `/api/cashflow/projection[/recompute]`,
`/api/anomalies[/{id}/ack|/detect]`, `/api/insights/{latest|daily|regenerate}`,
`/api/budgets[/{id}|/copy]`.

## Schedulers (13 loops + watchdog)

`full-sync` 6h, `health-check` 1h, `categorize-pending` 15min,
`promote-rules` 03h BRT, `detect-recurring` 02h BRT, `project-cashflow` 03h + pós-sync,
`cashflow-low-alert` 09h, `detect-anomalies` 1h, `daily-brief` 09h + catchup,
`weekly-brief` sáb 10h, `budget-check` 08h, `catchup-on-start` boot, `watchdog` 2min.

## Custos reais

- Pluggy prod: R$ 5-15/mês
- Claude (Haiku ~3k tx/mês + Sonnet 1 brief/dia, cache 90%): ~R$ 0,70/mês
- **Total: R$ 6-16/mês**

## Próximos passos sugeridos (opcionais)

- Investimentos via Pluggy `/investments` + cruzamento com [[b3-analyzer]] / [[dividend-portfolio]]
- PIX por chave (categorização auto quando descrição contém chave conhecida)
- Sazonalidade longa na projeção (13º, IPVA, IPTU)
- Parcelas futuras de cartão via Pluggy `/credit-installments`
- PWA mobile

Links: [[ecommerce-agent]] é a referência arquitetural mais próxima.
