---
name: claude-usage-dashboard
description: Pacote claude-usage e dashboard de tokens (porta 8520) que rastreia uso da API Anthropic em todos os projetos
metadata: 
  node_type: memory
  type: project
  originSessionId: 623673bf-bdbf-40a5-99af-ca4ffd7d7b0d
---

Sistema unificado de tracking de tokens Anthropic em todos os projetos do workspace.

**Pacote:** `/home/ubuntu/claude-usage/` (instalável via `pip install -e .`)
- `claude_usage/tracker.py` — `TrackedAnthropic` / `TrackedAsyncAnthropic` (wrappers transparentes) + `record_usage()` helper
- `claude_usage/db.py` — SQLite em `~/.claude-usage/usage.db`, tabela `usage` com ts/project/model/feature/tokens/cost
- `claude_usage/pricing.py` — preços por modelo (Opus/Sonnet/Haiku) com cache_read=10%, cache_creation=125%
- `claude_usage/agents_catalog.py` — catálogo estático de jobs/features por projeto (atualizar quando adicionar novo agente)
- `claude_usage/dashboard.py` — Streamlit com 3 abas (Overview, Agentes, Drill-down)

**Dashboard:** `http://localhost:8520` — systemd `claude-usage-dashboard.service` (enabled).

**Integrações ativas:**
- `email-agent/agent/config.py:get_ai_client()` → TrackedAsyncAnthropic
- `ecommerce-agent/agent/parser.py:OrderParser.__init__` → TrackedAsyncAnthropic (passa `raw_client` p/ Instructor)
- `ecommerce-agent/agent/orchestrator.py:_build_anthropic_client` → cliente compartilhado por instância
- `ecommerce-agent/agent/ai_insights.py` → TrackedAnthropic
- `followup-agent/nlu.py:_build_client` → TrackedAnthropic
- `b3analyzer/.../rag_engine.py` + `macro_insights.py` → TrackedAnthropic
- `dividend-portfolio/.../ai_analysis.py:_get_client` → TrackedAnthropic

**Why:** O usuário pediu visibilidade sobre quanto cada projeto gasta em Claude (tokens/dia/semana/mês/ano) e quais features/jobs consomem mais.

**How to apply:** Ao criar novas chamadas Anthropic em qualquer projeto, prefira importar `TrackedAnthropic`/`TrackedAsyncAnthropic` (com `try/except` fallback). Use `default_feature=` no construtor ou `_feature=` por chamada. Para libs externas (Instructor) que precisam do client puro, usar `.raw_client`. Toda integração tem fallback silencioso se `claude_usage` não estiver instalado, então não quebra nada.

**Pendente:** o `ecommerce-agent.service` precisa de restart pra carregar as mudanças (`kill <PID>` que o systemd reinicia em 30s).

Relacionado: [[ecommerce_alertas_cfo]] (jobs do scheduler que aparecem no catálogo).
