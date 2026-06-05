---
name: ecommerce-alerta-enviado-falso
description: "ecommerce-agent — log 'intraday-update enviado' é falso positivo: loga sucesso sem checar se o sendMessage do Telegram deu certo"
metadata: 
  node_type: memory
  type: project
  originSessionId: b988fa55-62a8-429d-bc3c-83c51c398212
---

# Alertas Telegram — "enviado" é falso positivo

`_run_intraday_update` (`scheduler.py:3404`) e jobs similares logam
`intraday-update [Nh]: enviado` **incondicionalmente**, sem checar o retorno do
`sendMessage`. Quando a rede/DNS está fora, o log mostra
`ERROR | Erro ao enviar Telegram [...]: Temporary failure in name resolution`
seguido de `INFO | ... enviado` — ou seja, "enviado" no log NÃO garante entrega.

**Caso real (02/06/2026):** host perdeu resolução de DNS ~17:42 até ~07:18 do dia
seguinte (outage de infra, não bug). Relatórios 18h/21h/23h logaram "enviado" mas
NÃO chegaram no Telegram; o último entregue foi o das 15h. GMV congelou em 7.682
porque o `ml-orders-sync` também falhava.

**RESOLVIDO (jun/2026):**
1. `_run_intraday_update` e `_run_daily_sales_report` agora capturam o retorno do
   `_notify` (=`send_text`, que retorna bool) e **só marcam `*_last_sent` + logam
   "enviado" se a entrega foi confirmada**; senão logam WARNING "FALHA no envio".
   `_run_intraday_update` passou a retornar bool.
2. Novo loop `_loop_retry_missed_reports` (task `retry-missed-reports`, a cada 10
   min): reprocessa horários passados de hoje ainda não marcados → reenvia assim que
   a rede volta, sem depender de restart. Intraday é cumulativo, então reenvia só o
   horário mais recente perdido e **suprime os anteriores** (marca como sent) p/ não
   despejar relatórios defasados. Daily report (09h) também coberto.

Como o estado só é marcado na entrega confirmada, o catch-up no restart e o novo
loop convergem naturalmente. Lint/baseline inalterados; py_compile OK.

## Auditoria completa dos ~35 jobs (jun/2026, commit d752614)

O mesmo bug existia em vários outros jobs além do intraday/daily. Auditados e corrigidos em 3 grupos:

**Grupo A — perda silenciosa (marcava estado sem confirmar _notify):** `check-nfe-sem-custo` (hash+last_sent, cooldown 7d), `order-anomalies` queda/spike (cooldown 4h/2h), `order-anomalies` alto valor + `check-non-full-orders` (avançavam `last_seen_*_id` ANTES de enviar → pedido perdido pra sempre; agora avança por item enviado e dá `break` na 1ª falha), `check-ruptura-ml` (cooldown 24h). Padrão: `ok = await self._notify(...)`; só marca estado se `ok`.

**Grupo B — escape MarkdownV2 em parse_mode legado.** O cliente (`telegram_client._send_text_to`) usa `parse_mode="Markdown"` (legado), que **NÃO** suporta escape com barra — `\.`, `\!`, `\(`, `\)`, `\-`, `\+`, `\$` renderizam a **barra invertida LITERAL** na tela (confirmado empiricamente 04/06/2026: linha `Por Empresa \(ML\)` apareceu com as barras). No legado só `* _ [ \`` são especiais; o resto é literal e NÃO deve ser escapado. Removidos os escapes de ~17 mensagens (daily/intraday/weekly/monthly summary, promo, nfe-status/custo/not-synced/sem-custo, saldo, havan, ml-catalog, ruptura, ima-depletion, watchdog). Funções `_esc` que faziam escape V2 viraram no-op (dado com `*`/`_` raro; o fallback do cliente reenvia em texto plano no erro de parse). **NÃO reintroduzir escape estilo MarkdownV2.**

**Grupo C — log honesto:** jobs que só logavam "enviado" sem persistir estado agora logam ERROR quando `_notify` retorna False (check-avencer, check-saldo, nfe-status, nfe-custo, commission, ml-catalog, nfe-sync-integrity, weekly/monthly-summary). `_alert` (wrapper de erro) não retorna bool — best-effort, sem checagem.

Ver [[ecommerce-money-release-date]] (mesmo padrão aplicado ao check-provisao).
