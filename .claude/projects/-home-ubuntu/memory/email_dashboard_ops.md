---
name: email-dashboard-ops
description: "email-agent — operar o dashboard Streamlit: porta 8502, restart manual vs refresh de cache, rate limit do Google Sheets"
metadata: 
  node_type: memory
  type: project
  originSessionId: cdca63bc-a576-479d-afdf-799ffd73b806
---

Operação do dashboard Streamlit do email-agent (Motorleads Internacional).

**Processo / launch:** roda em `agent/dashboard/app.py` na **porta 8502**, launch MANUAL (não há systemd — diferente do ecommerce/finance). Comando:
```
cd /home/ubuntu/email-agent && nohup /usr/bin/python3.12 -m streamlit run agent/dashboard/app.py --server.port 8502 --server.headless true --browser.gatherUsageStats false > dashboard_8502.log 2>&1 & disown
```
PID via `ss -tlnp | grep 8502`. (Há outros Streamlit no host: b3=8501, dividend=8506, claude-usage=8520.)

**Restart vs Refresh (importante):** o botão **"🔄 Forçar Atualização"** na sidebar só chama `st.cache_data.clear()` + `clear_sheets_cache()` — relê as planilhas usando o **código que já está em memória**. NÃO reimporta módulos `.py`.
- Mudança em **dado** (planilha) → basta o refresh (cache TTL 5 min de qualquer forma).
- Mudança em **código** (qualquer `.py`) → **tem que reiniciar o processo** do Streamlit (kill + relançar). Senão a alteração não reflete. **Why:** confundir isso faz parecer que o fix "não funcionou". Confirmado 08/jun/2026 (fix da projeção só apareceu após restart). [[feedback-testar-so-o-que-mudou]]

**Rate limit Google Sheets:** quota de leitura é **60 read requests/min/user** (HTTP 429 `RATE_LIMIT_EXCEEDED`). Ao investigar lendo planilhas direto (fora do cache do app), agrupar com `batchGet` (1 request p/ vários ranges) e usar retry com backoff ~65s. `read_all_countries` + `read_payment_history` juntos fazem ~14 chamadas e podem estourar a cota se repetidos em sequência.

**Aba Receitas** (por país): tem cabeçalho real na linha 1 (índice 1), col **M (índice 12) = "Fatura"** (nº; pode listar vários separados por "/"). Layout de colunas é fixo/uniforme entre países (constantes `_COL_*` em `agent/payments/utils.py`). Brasil NÃO tem aba "Receitas" (erro 400 conhecido, ignorado).
