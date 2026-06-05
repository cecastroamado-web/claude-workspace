---
name: Alertas CFO ecommerce-agent — jobs do scheduler
description: Inventário dos jobs automáticos do scheduler.py do ecommerce-agent (alertas, relatórios e syncs)
type: project
originSessionId: c55d1021-f78f-437e-a9da-511f2b2ea0d7
---
Jobs automáticos no `agent/scheduler.py`:

- **Promo ML** (5min): detecta promoções derrubadas (rate lock + None safe em erro de API)
- **Vendas diárias**: 09h relatório de ontem; 12h/15h/18h/21h updates intraday vs ontem
- **check-avencer** 07h45: títulos vencendo em ≤2 dias
- **check-saldo** 10h: saldo bancário < R$30k por empresa via `sheets.get_summary`
- **weekly-summary** seg 08h: resumo ML + B2B vs semana anterior
- **monthly-summary** dia 1 09h: fechamento mensal + YoY
- **order-anomalies** 30min (08-22h): queda 0 pedidos em 3h / spike 2× / pedido > R$1.500
- **check-nfe-status** 2h: NF-e sem `situacao=6` há > 12h via `bling.list_nfe_all()`
- **check-commission** 09h30: variação > 1pp comissão ML
- **check-ml-catalog** 2h: itens ML que ficaram pausados
- **check-ima-depletion** 08h15: FIFO sobre aba "Saídas 2026" → quando 88mm Brasil esgotar, insere `ima_prices` China + propaga `product_costs` automaticamente + alerta Telegram
- **imas-fetch-failure** (não é job, é guard em `agent/api.py`): `_fetch_imas_data` conta falhas consecutivas e dispara Telegram após 3× com cooldown 6h. Mensagem diferenciada p/ `invalid_grant` (manda `python3 scripts/setup_drive_oauth.py`). Sem isso, o token Drive expirado deixava `pipeline_producible=0` silenciosamente.
- **check-nfe-sync-integrity** 09h45: conta `ml_nfe` com `data_emissao` NULL/vazia sincronizadas nos últimos 30 dias. Alerta Telegram se > 5% ou > 50 NFs por empresa. Detecta o tipo de erro de sync que escondia 30% das NFs do relatório por filial em mai/2026 (ver [data_emissao gotcha](ecommerce_ml_nfe_data_emissao.md)).

**Convenções:**
- Thresholds são constantes no topo de `scheduler.py` (`SALDO_MINIMO`, `ALTO_VALOR_PEDIDO`, etc.)
- State efêmero em `self._alert_state` (cooldowns gap/spike, `last_seen_order_id`, `catalog_paused_ids`)
- Tabela `scheduler_state` no DB rastreia `last_sent` por job para evitar duplicatas após restart
- Catch-up: scheduler roda report perdido 90s após subir (daily 09h + intraday 12/15/18/21h)
- Sync ML usa `date_approved` (não `date_created`) para bater com painel ML
