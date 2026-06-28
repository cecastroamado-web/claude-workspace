---
name: host-memoria-swap-thrashing
description: Host fica sem RAM (8 serviços/5 projetos) → swap thrashing congela processos e desalinha timers; mitigações aplicadas jun/2026
metadata: 
  node_type: memory
  type: project
  originSessionId: 85b1fdaa-2bbf-46c3-bef8-79b101b760fb
---

**Causa raiz de "suspensão/salto de relógio" do host** (investigado 26/jun/2026): NÃO é suspensão nem relógio (uptime 12 dias contínuos, NTP sincronizado). É **pressão de memória → swap thrashing**.

O host tem **7,7 GB RAM** e roda 8 serviços systemd de 5 projetos + Claude Code. Consumo (jun/2026): **b3-alerts 2,4 GB** (vilão, vaza até OOM), email-agent 1,2 GB, email-dashboard 867 MB, ecommerce-agent ~300 MB, b3-dashboard 131 MB, followup 57 MB, claude.exe ~500 MB. Soma > RAM → swap satura → kernel congela processos durante paging por segundos/minutos.

**Sintomas que isso causa** (não confundir com bug de código):
- Gaps nos logs (processo congelado não loga).
- `asyncio.sleep` longos atrasam → timers diários (havan 06h, briefing 06h30, envios 11h) **perdem a janela** → alertas não saem. Por isso os jobs precisam de **retry-missed/catch-up** (já implementados: briefing, envios, havan-sync).
- ConnectError/timeouts de rede durante o congelamento.
- **OOM killer** mata processos: matou `b3-alerts` em 24/jun (4,2 GB).

**Mitigações aplicadas (26/jun/2026, via sudo — drop-ins systemd + swap):**
- `/etc/systemd/system/b3-alerts.service.d/override.conf`: **MemoryHigh=1G + MemoryMax=1300M** → throttle + mata/reinicia limpo ao vazar (Restart=on-failure). Vira "restart automático ao vazar".
- `/etc/systemd/system/ecommerce-agent.service.d/override.conf`: **OOMScoreAdjust=-500** → nunca é a vítima preferencial do OOM.
- **Swap 2 GB → 4 GB**: criado `/swapfile2` (2 GB), persistido em `/etc/fstab`. Pós-fix: RAM disp. ~3,5 GB, swap livre ~1,1 GB.

**Mitigações 2ª rodada (26/jun, aplicadas):**
- ~~Dashboards ociosos parados~~ → **REVERTIDO 27/jun (NÃO repetir).** Parei `email-dashboard` (8502) e `b3-dashboard` (8501) achando que eram ociosos, mas **o CFO USA os dois** (acessa via `http://192.168.2.2:8501` e `:8502`). Religados e **reabilitados no boot**. ⚠️ **8501 (b3-dashboard) e 8502 (email-dashboard) são dashboards EM USO — não parar.** Com os outros fixes (swap 4GB + b3-alerts limitado) há folga: religados, ainda sobram ~3GB de RAM disponível. Se precisar liberar RAM no futuro, cortar OUTRA coisa ou perguntar antes — não esses dashboards. (dividend/followup/claude-usage-dashboard são 3MB, irrelevantes.)
- **Vazamento b3-alerts investigado:** NÃO é bug de 1 linha — é acúmulo inerente do **yfinance+pandas** (daemon escaneia ~80 tickers IBOV × 1y de dados/ciclo via `b3 alert daemon --interval 15`). O `daemon.py` já mitiga (limpa `yf.shared._DFS`, `gc.collect()`, `_clear_ctx_cache()`) e tem **watchdog interno** (`MAX_RSS_MB`, restart limpo via `os.execv`, estado em disco). Fix: **MAX_RSS_MB 2048→1300** (`b3analyzer/daemon.py`, commit c25121a) + systemd `MemoryHigh=1300M`/`MemoryMax=1500M` (rede de segurança ACIMA do watchdog interno, pra o restart limpo agir primeiro).
- **Resultado:** RAM disponível 2,5 GB → **4,8 GB**; swap livre 2,4 GB. Thrashing resolvido.

**Se voltar a faltar RAM:** reduzir o universo de scan do b3 (IBOV inteiro × 1y é caro) ou baixar mais o MAX_RSS_MB; parar email-agent dashboard de vez se não for usado.

**Crash loop do b3-alerts (27/jun, resolvido) — era FILE DESCRIPTORS, não memória:** o b3 reiniciava a cada ~35 min com `Main process exited status=1/FAILURE`. Causa: o daemon **vaza file descriptors** (yfinance/requests sockets + ChromaDB-pyo3 + ~56 FDs `/sys/.../cpu` por ciclo). O **soft limit padrão era 1024** (hard 524288, mas o processo respeita o soft) → ao estourar: `OSError [Errno 24] Too many open files` → `pyo3_runtime.PanicException` (ChromaDB backend Rust, 548k docs) → exit 1. São DOIS vazamentos independentes: memória (watchdog `os.execv` a 1300MB já cobre) e FDs (este). Fix: **`LimitNOFILE=65536`** no drop-in systemd → crash loop cessou (0 exit-1 após). Paliativo — o vazamento de FD continua, mas o `os.execv` periódico reseta antes de chegar a 65k. **Caça ao leak de FD (27/jun):** fontes confirmadas via `/proc/<pid>/fd` — **19 sockets CLOSE_WAIT** (yfinance, HTTP não fechado), ChromaDB `.bin` reabertos (~13×) e `/sys/.../cpu` re-detectado (~13×). Tentei fechar `yf.data.YfData()._session` no cleanup do daemon, mas **QUEBRA o `history()` seguinte** (singleton não recria a session → `'NoneType' object is not subscriptable`) — revertido (commit 35d7720, deixei nota anti-regressão no daemon). Conclusão: o leak de FD **não tem fix de código seguro** (internals do yfinance/chromadb); fica **contido por infra** — `LimitNOFILE=65536` + `os.execv` a cada ~35min resetam os FDs muito antes do teto. Estável.

**Log gigante do b3 (27/jun, resolvido):** `/home/ubuntu/b3analyzer/b3analyzer.log` chegou a **1,5 GB** — `StandardOutput`+`StandardError=append:...` (nunca rotaciona) capturando o logger em nível DEBUG (~80 tickers × correlações × loop 15min). Fix: `truncate -s 0` + **`/etc/logrotate.d/b3-alerts`** (size 100M, rotate 3, compress, copytruncate). Pendente opcional: baixar nível de log do daemon DEBUG→INFO.

Relacionado: fixes de auto-cura dos timers em [[ecommerce-briefing-matinal-cfo]] e no scheduler (retry-missed cobre briefing/envios/havan).
