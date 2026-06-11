---
name: ecommerce-havan-transito-estoque
description: Havan em-trânsito — relatório de recebimento da Havan atrasa vs estoque; decisão pendente de trocar timer 15d por detecção via aumento de estoque
metadata: 
  node_type: memory
  type: project
  originSessionId: e2af3fe4-597b-47f4-bda5-9f55613c3975
---

Painel Havan, conceito "em trânsito" tem DUAS fontes no `ecommerce-agent/agent/api.py`:
- **Qtd "X itens em NF não entregue"** = timer cego: itens de NF onde `data_emissao + 15 dias > hoje` (`_havan_em_transito_por_sku`, const `_HAVAN_TRANSITO_DELAY_DAYS = 15`, ~`api.py:8425/8508`). Não reflete entrega real.
- **R$ "Em trânsito (não recebido)"** = conciliação mensal `nao_entrada` = `nosso_faturamento − havan_recebido`, onde `havan_recebido` vem de `recebido_mensais` do snapshot (relatório oficial de recebimento da Havan).

**Problema (11/jun/2026):** Havan recebeu 3 NFs de 01/06 (3.210 itens) hoje, **estoque subiu** no sync manual das 11:55h — MAS `recebido_mensais['06/2026'] = 0` para todos os produtos. O relatório de recebimento da Havan **atrasa** em relação ao estoque; o estoque é o sinal que chega primeiro. As 3 NFs ainda contam como em-trânsito (01/06 + 15d = 16/06 > hoje).

**Decisão do CFO:** trocar o timer de 15d por **detecção via aumento de estoque** (em-trânsito cai quando o estoque sobe). Modelo desenhado: `recebido_real[sku]` = baseline + soma dos acréscimos de estoque entre syncs (quedas = vendas, ignoradas); `em_transito_qtd = max(0, faturado_total[sku] − recebido_real[sku])`; seed único = faturado total de hoje (tudo já entregue → zera hoje); SKU novo futuro entra com recebido=0. Tabela nova tipo `havan_recebido_estoque(sku, recebido_qtd, last_estoque)`. Manter conciliação mensal `por_mes` como lente separada ("relatório oficial Havan, atrasa").

**PENDENTE — retomar 12/jun/2026:** o usuário pediu pra ESPERAR o sync automático das 06h de amanhã antes de codar. Checar `recebido_mensais` de junho no snapshot: se populou, conciliação R$ se ajusta sozinha e talvez não precise mexer; se continuar 0, IMPLEMENTAR o modelo por estoque. Nenhuma alteração de código feita ainda.

**Verificação agendada (cron LOCAL, não nuvem):** script `ecommerce-agent/scripts/check_havan_recebido_junho.py` agendado via crontab `30 7 12 6 *` (12/jun 07h30 local, após sync 06h). Lê o DB, checa `recebido_mensais['06/2026']` e manda Telegram: ✅ se populou / ⚠️ se ainda 0 (pedindo p/ implementar). É one-shot — remove a própria linha do crontab ao rodar. Log em `/home/ubuntu/havan_check.log`. NOTA: rota de nuvem (skill /schedule) foi descartada porque agente na nuvem não acessa o DB local de runtime.

Disparo manual do sync: `POST /api/havan/sync` (mesmo job das 06h). Token JWT mintável com `JWT_SECRET` default (não setado no .env). Ver [[ecommerce-havan-dashboard]], [[ecommerce-havan-backlog-faturamento-transito]].
