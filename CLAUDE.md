# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This home directory contains 6 independent Python projects, each in its own subdirectory. All use Python 3.12, `uv`/`hatchling` as build system (except `followup-agent` which uses `requirements.txt`), and `ruff`+`mypy` for linting/typing.

---

## Projects

### 1. `ecommerce-agent` — E-commerce Operations Agent
Order pipeline for Starlink accessories: Telegram → Claude AI → Bling ERP (NF-e) → Banco Inter (boleto) → Google Sheets → Telegram (PDFs).

**Run:**
```bash
# Managed by systemd — do NOT run manually with python/nohup
systemctl status ecommerce-agent.service
journalctl -u ecommerce-agent.service -n 50 --no-pager
# To restart: kill the PID, systemd restarts automatically in ~30s
ss -tlnp | grep 8510 | grep -oP 'pid=\K[0-9]+'
```

**Ports:** `8504` = Telegram webhook, `8510` = FastAPI dashboard + React UI

**Frontend rebuild:**
```bash
cd ecommerce-agent/dashboard-ui && npm run build
```

**Dev commands:**
```bash
cd ecommerce-agent
pip install -e ".[dashboard,dev]"
ruff check .
mypy agent/
pytest tests/ -v
pytest tests/test_bling.py -v   # single test file
```

**Key architecture:**
- `agent/orchestrator.py` — Multi-step order pipeline (Claude tool_use → Bling → Inter → Sheets)
- `agent/scheduler.py` — All automated alerts/cron jobs (CFO alerts, ML sync, NF-e checks)
- `agent/api.py` — Dashboard REST API (cashflow, profitability, ML NF-e, DIFAL)
- `agent/metrics.py` — Revenue queries (ML orders, Bling NF-e, NFS-e)
- `agent/db.py` — SQLite async (aiosqlite); tables: `ml_orders_sync`, `bling_nfe`, `bling_nfse`, `ml_nfe`, `ima_prices`, `product_costs`, `scheduler_state`
- `companies.json` — Multi-tenant config: XConnect (Lucro Presumido) + Aviation (Simples Nacional), each with own Bling/Inter credentials
- `agent/config.py` — All env vars and constants

**Revenue rules (critical):**
- ML revenue: `ml_orders_sync` WHERE `status='paid'` AND NOT cancelled
- Bling NF-e revenue: `bling_nfe` WHERE `situacao=6`, exclude EBAZAR recipients
- NFS-e revenue: `bling_nfse` WHERE `situacao=1` only (situacao=3 = anulada, NOT revenue)
- Never use `bling_orders` table as revenue source

**Scheduler alert rules (`agent/scheduler.py` — critical):**
- **Telegram uses legacy `parse_mode="Markdown"`** (`agent/telegram_client.py`), NOT MarkdownV2. Backslash escapes render LITERALLY on screen — never write `\.`, `\!`, `\(`, `\)`, `\-`, `\+`, `\$` in alert text. Only `* _ [ \`` are special in legacy; leave everything else unescaped (the client auto-falls back to plain text on a parse error).
- **`_notify` / `send_text` return `bool`** (True only if Telegram delivered). A job must check it BEFORE marking any persisted state — `set_scheduler_state(*_last_sent)`, cooldowns in `_alert_state`, or advancing `last_seen_*_id`. Pattern: `ok = await self._notify(...); if ok: <mark state>` else `logger.error(...)`. Logging "enviado" unconditionally is a false positive that silently drops alerts during a network/DNS outage.
- When iterating and sending per-item (high-value orders, non-full orders), advance the `last_seen_*_id` pointer per successfully-sent item and `break` on the first failure — never advance past an unsent item, or it's lost forever.
- Provisão alert (`_run_check_provisao`) and dashboard (`/api/provisao`) share one source of truth: the alert calls `get_provisao()`. Model is no-anticipation (money_release_date real do MP). Don't reintroduce the old anticipation/média-diária logic.

---

### 2. `email-agent` — Gmail + Telegram + Claude AI
Monitors multiple Gmail accounts, analyzes emails with Claude, notifies via Telegram, drafts replies.

**Run:**
```bash
cd email-agent && python main.py
```

**Dev commands:**
```bash
cd email-agent
pip install -e ".[dev]"
ruff check .
mypy agent/
pytest tests/ -v
```

**Key architecture:**
- `agent/analyzer.py` — Claude-powered email classification (importance, urgency, meeting extraction)
- `agent/message_handler.py` — Telegram command handler using Claude tool_use
- `agent/gmail.py` — Async IMAP listener per account (aioimaplib)
- `agent/tools.py` and `agent/reports.py` — Tool definitions with prompt caching (`cache_control: {"type": "ephemeral"}`)
- `accounts.json` — Per-account Gmail + Calendar OAuth tokens (see `accounts.example.json`)
- `agent/financeiro/` — Financial module reading Google Sheets (10 bank accounts, 6 countries)
- `agent/scheduler.py` — Scheduled jobs: `payment_scheduler` (daily) + `international_results_scheduler` (day 2 of each month at 09h05, sends operational result per country vs. 2025 average via Telegram)
- `agent/dashboard/app.py` — Streamlit dashboard with: DRE, IVA panel, Repasse de Mídia (wire flow to Reweb Corp + Sistemas Reweb USD 5,152/month from Feb/26), Fluxo de Wire por Destino (Reweb/Wire Brasil/Consultores/Outros), Compromissos por país
- `agent/payments/commitments_reader.py` — Reads "Compromissos" tab (cols A-F: Nome, Pacote, Valor, Dia Vcto, Ativo, Condição) from each country's spreadsheet

**Dashboard wire classification (app.py):**
- `_WIRE_SENDERS = {"Colombia", "Mexico", "Ecuador"}` — countries that send wire to Reweb Corp for media
- Wire destinations: `wire_midia` (Reweb/RW Corp), `wire_brasil` (CR2/"Wire Brasil"), `wire_consultores` (Consulexpress/Partner Share), `wire_outros` (remaining)
- Sistemas Reweb: USD 8,703/month until Jan/2026; USD 5,152/month from Feb/2026 (`_reweb_sistemas()`)

**AI features:** Prompt caching on system prompts, streaming with progressive Telegram edits (0.8s interval), history summarization via Haiku when >12 messages, Extended Thinking for complex financial reports.

---

### 3. `b3analyzer` — B3 Stock Market CLI
CLI for Brazilian stock analysis: technical, fundamental, options, sentiment, RAG insights.

**Run:**
```bash
cd b3analyzer
b3 <command>          # after pip install -e .
b3 analyze PETR4
b3 screen --dy 6
b3 dashboard          # starts Streamlit on port 8501
```

**Services:** `b3-alerts.service` (daemon), `b3-dashboard.service` (Streamlit)

**Dev commands:**
```bash
cd b3analyzer
pip install -e ".[dashboard,rag,sentiment]"
ruff check .
mypy .
pytest tests/ -v
```

**Key architecture:**
- `b3analyzer/cli/` — Click CLI entry point (`b3analyzer.cli:cli`)
- `b3analyzer/analysis/` — Technical (pandas-TA), fundamental, sentiment, correlation, volatility modules
- `b3analyzer/data/` — Brapi.dev premium client + yfinance fallback + SQLite TTL cache
- `b3analyzer/data/crawl_client.py` — Crawl4AI web scraping (InfoMoney, Valor Econômico)
- `b3analyzer/data/vector_store.py` — ChromaDB with `_hash_embed` (NOT sentence-transformers — causes SEGFAULT with FinBERT/OpenBLAS conflict)
- `b3analyzer/analysis/rag_engine.py` — RAG narratives via Claude Haiku + ChromaDB context
- Config: `~/.b3analyzer/config.json`

---

### 4. `dividend-portfolio` — Dividend Growth Portfolio Engine
Screening, Moat/Munger analysis, Kelly Criterion, Monte Carlo simulation, FIRE tracking for Brazilian stocks.

**Run:**
```bash
cd dividend-portfolio
div <command>          # after pip install -e .
div screen --dy 5
div analyze TAEE11
div portfolio
div simulate
streamlit run dividend_portfolio/dashboard/app.py  # port 8506
```

**Dev commands:**
```bash
cd dividend-portfolio
pip install -e ".[dashboard,dev]"
ruff check .
mypy .
pytest tests/ --cov=dividend_portfolio -v
```

**Key architecture:**
- `dividend_portfolio/cli/` — Click CLI entry point (`dividend_portfolio.cli:cli`)
- `dividend_portfolio/analysis/` — Screening, valuation (DCF/DDM/Graham), dividend safety (13-metric score), backtest, FIRE tracker, tax impact (Lei 15.270/2025)
- `dividend_portfolio/portfolio/` — Kelly Criterion, HRP, Mean-CVaR optimization
- `dividend_portfolio/data/` — Brapi.dev + yfinance + FRED macro + CEI custody import
- `dividend_portfolio/data/provento_store.py` — Dividend history CRUD
- Config: `~/.dividend-portfolio/config.json`
- **Note:** yfinance DY for B3 is unreliable — always compute from real 12-month dividend history

---

### 5. `followup-agent` — Telegram Follow-up Manager
Natural language follow-up reminders via Telegram with voice support.

**Run:**
```bash
cd followup-agent && python agent.py
```

**Dev commands:**
```bash
cd followup-agent
pip install -r requirements.txt
# No formal test suite
```

**Key architecture:**
- `agent.py` — Main loop: Telegram listen → NLU → action → store
- `nlu.py` — Claude Haiku NLU with prompt caching, returns structured JSON actions
- `store.py` — JSON persistence (`DATA_DIR/followups.json`)
- `telegram.py` — Async Telegram Bot client
- `transcriber.py` — Optional Groq Whisper audio transcription
- `memory.py` — Optional Mem0AI persistent memory
- Scheduled reminders at 09h, 14h, 18h

---

### 6. `finance-agent` — Personal Finance Dashboard (PF)
Personal finance dashboard: Pluggy (Open Finance Brasil) auto-sync → Claude AI categorization/insights → Telegram alerts → React/FastAPI dashboard.

**Run:**
```bash
cd finance-agent && python3 main.py
# systemd unit file exists (finance-agent.service) but is NOT installed yet — run manually
```

**Ports:** `8512` = FastAPI dashboard + React UI (JWT login), `8513` = Pluggy webhook (expose via ngrok in dev)

**Frontend rebuild:**
```bash
cd finance-agent/dashboard-ui && npm run build
```

**Dev commands:**
```bash
cd finance-agent
pip install -e ".[dev]"
ruff check .
mypy agent/
python3 scripts/setup_pluggy.py --sandbox   # validate Pluggy credentials
# No formal test suite yet
```

**Key architecture (16 modules in `agent/`):**
- `agent/api.py` — FastAPI ~30 REST endpoints + JWT auth + SPA static files
- `agent/scheduler.py` — 13 async loops + watchdog + catchup on restart
- `agent/sync.py` — Pluggy → DB incremental sync (every 6h + webhook) + credit-card payment dedup (regex matching, not counted as expense)
- `agent/categorizer.py` — Rules first → Claude Haiku batch of 20 + prompt caching; inline corrections auto-promote to rules after 3 identical confirmations
- `agent/cashflow.py` — D+90 projection (balance + recurrings + CC invoice + DOW average)
- `agent/recurring.py` — Monthly/biweekly/weekly recurrence detector (z-score)
- `agent/anomalies.py` — 4 detectors: outlier (z-score), duplicate, overspend, new_merchant_large
- `agent/ai_insights.py` — Daily/weekly brief with Sonnet + 12h cache, sent via Telegram at 09h
- `agent/budgets.py` — Monthly budgets with 80%/100% alerts + rollover
- `agent/db.py` — SQLite async (aiosqlite WAL), 13 tables
- `agent/webhook.py` — Separate FastAPI on :8513, HMAC SHA256 validation
- `dashboard-ui/` — React 19 + Vite 6 + TS + Tailwind v4 + TanStack Router/Query/Table; 7 routes, 13 feature components

**Critical rules:**
- Keep `PLUGGY_BASE_URL=https://api.sandbox.pluggy.ai` until the flow is validated (sandbox credentials: `user-ok`/`password-ok`)
- Credit-card invoice payments are deduped in `sync.py` — never count them as expenses
- See `PLAN.md` for full endpoint list and scheduler job inventory

---

## Common Patterns Across Projects

**Anthropic SDK (0.84.0):**
- Prompt caching: system prompt as `list[dict]` with `cache_control: {"type": "ephemeral"}`
- Streaming: `async with client.messages.stream() as stream: async for text in stream.text_stream:`
- Extended Thinking: `thinking={"type": "enabled", "budget_tokens": N}` — `max_tokens` must exceed `budget_tokens`
- Telegram streaming: `sendMessage` → save `message_id` → `editMessageText` every 0.8s

**Smart model routing:** Simple queries → `claude-haiku-4-5-20251001`, complex → `claude-sonnet-4-6`

**Mem0AI memory:** Optional in email-agent, ecommerce-agent, followup-agent — always wrapped in try/except so failure doesn't break main flow.

**Environment files:** Each project has its own `.env`. See `.example` files for required keys.
