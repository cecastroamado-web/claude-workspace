# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This home directory contains 5 independent Python projects, each in its own subdirectory. All use Python 3.12, `uv`/`hatchling` as build system (except `followup-agent` which uses `requirements.txt`), and `ruff`+`mypy` for linting/typing.

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

## Common Patterns Across Projects

**Anthropic SDK (0.84.0):**
- Prompt caching: system prompt as `list[dict]` with `cache_control: {"type": "ephemeral"}`
- Streaming: `async with client.messages.stream() as stream: async for text in stream.text_stream:`
- Extended Thinking: `thinking={"type": "enabled", "budget_tokens": N}` — `max_tokens` must exceed `budget_tokens`
- Telegram streaming: `sendMessage` → save `message_id` → `editMessageText` every 0.8s

**Smart model routing:** Simple queries → `claude-haiku-4-5-20251001`, complex → `claude-sonnet-4-6`

**Mem0AI memory:** Optional in email-agent, ecommerce-agent, followup-agent — always wrapped in try/except so failure doesn't break main flow.

**Environment files:** Each project has its own `.env`. See `.example` files for required keys.
