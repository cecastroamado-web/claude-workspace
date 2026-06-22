---
name: ecommerce-havan-transito-estoque
description: Havan em-trânsito — relatório de recebimento da Havan atrasa vs estoque; decisão pendente de trocar timer 15d por detecção via aumento de estoque
metadata: 
  node_type: memory
  type: project
  originSessionId: e2af3fe4-597b-47f4-bda5-9f55613c3975
---

## ✅ RESOLVIDO (19/jun) — em-trânsito só na JANELA ABERTA (mês corrente + anterior); fim do trânsito fantasma
Bug (CFO 19/jun): 650 haste + 300 case apareciam "em trânsito como se faturado". Causa: NFs ANTIGAS já entregues (NF 561 jan/2026 — cujos 300 case foram DEVOLVIDOS e reemitidos na 564) figuravam como trânsito porque o **relatório de recebimento da Havan tem buraco PERSISTENTE em meses fechados** (haste jan=0; case jan=300 de 600). Como `_havan_em_transito_por_sku` fazia `faturado(all-time) − recebido(all-time)`, o buraco virava fantasma pra sempre. **Re-coleta NÃO corrige** (a lacuna está no portal, não é glitch transitório — testado).
Fix (commit `693a44b`): `_havan_em_transito_por_sku` restringe faturado E recebido à **janela aberta = mês corrente + anterior** (cutoff = 1º dia do mês anterior). Meses fechados ficam **assentados** (NF entregue há semanas → não pode ser trânsito) e **imunes a buracos no relatório de meses antigos**. Só NFs ≤~60d podem estar a caminho do CD. `_havan_faturado_qtd_por_sku(conn, desde=)` + novo `_havan_recebido_recente_for_nf_sku(produtos, sku, cutoff_ym)` (soma recebido_mensais dos meses ≥ cutoff). Validado: haste 650→0, case 300→0; trânsito restante = só NFs de junho (581/586/588/589, reais). Mudança só de código (nada gravado no banco).
**Otimização pendente (CFO sugeriu, não feita):** o scraper ainda re-consulta jan→jun toda coleta; ideal congelar meses fechados em cache e buscar só o mês corrente (mais rápido + imune a glitch de mês fechado). A correção acima já blinda o em-trânsito; o cache é ganho de performance separado.

## ✅ RESOLVIDO (18/jun) — Opção B: em-trânsito = faturado − recebido (não precisou do modelo de estoque)
O relatório oficial de recebimento da Havan **voltou a atualizar** (junho populado: Slim 850, Haste 500, etc.) → o problema de 11/jun (relatório travado em 0) **se resolveu sozinho**. Então NÃO implementei o modelo de aumento-de-estoque (era workaround). Em vez disso (commit `3bf5f0e`):
- **`_havan_em_transito_por_sku(conn, produtos, ...)`** agora = **faturado (NFs válidas, `_havan_faturado_qtd_por_sku`) − recebido (`recebido_periodo`, match via `_havan_recebido_for_nf_sku`)**. Removido o timer 15d. Se o relatório atrasar, itens ficam corretamente em trânsito até o recebimento aparecer.
- **R$ em-trânsito** segue = **divergência** (nosso_total bling_nfe − recebido_valor; autoritativo p/ valor). `nao_entrada`/mês = divergência do mês (soma = 132.330). NÃO usar faturado_qty×atacado p/ R$ (não reconcilia: preço atacado do snapshot ≠ valor histórico da NF).
- Removida `_havan_em_transito_valor_por_mes` (FIFO×atacado, dead code). `_HAVAN_TRANSITO_DELAY_DAYS`/`_havan_atacado_for_sku` ficaram órfãos (inofensivos, não removidos).
- Validado: em_transito_qtd 2.975 (Cabo3m 1430, Haste 650, Case 300, Painel 500, CaboUSB 95); R$ 132.330.

---

Painel Havan, conceito "em trânsito" tem DUAS fontes no `ecommerce-agent/agent/api.py`:
- **Qtd "X itens em NF não entregue"** = timer cego: itens de NF onde `data_emissao + 15 dias > hoje` (`_havan_em_transito_por_sku`, const `_HAVAN_TRANSITO_DELAY_DAYS = 15`, ~`api.py:8425/8508`). Não reflete entrega real.
- **R$ "Em trânsito (não recebido)"** = conciliação mensal `nao_entrada` = `nosso_faturamento − havan_recebido`, onde `havan_recebido` vem de `recebido_mensais` do snapshot (relatório oficial de recebimento da Havan).

**Problema (11/jun/2026):** Havan recebeu 3 NFs de 01/06 (3.210 itens) hoje, **estoque subiu** no sync manual das 11:55h — MAS `recebido_mensais['06/2026'] = 0` para todos os produtos. O relatório de recebimento da Havan **atrasa** em relação ao estoque; o estoque é o sinal que chega primeiro. As 3 NFs ainda contam como em-trânsito (01/06 + 15d = 16/06 > hoje).

**Decisão do CFO:** trocar o timer de 15d por **detecção via aumento de estoque** (em-trânsito cai quando o estoque sobe). Modelo desenhado: `recebido_real[sku]` = baseline + soma dos acréscimos de estoque entre syncs (quedas = vendas, ignoradas); `em_transito_qtd = max(0, faturado_total[sku] − recebido_real[sku])`; seed único = faturado total de hoje (tudo já entregue → zera hoje); SKU novo futuro entra com recebido=0. Tabela nova tipo `havan_recebido_estoque(sku, recebido_qtd, last_estoque)`. Manter conciliação mensal `por_mes` como lente separada ("relatório oficial Havan, atrasa").

**PENDENTE — retomar 12/jun/2026:** o usuário pediu pra ESPERAR o sync automático das 06h de amanhã antes de codar. Checar `recebido_mensais` de junho no snapshot: se populou, conciliação R$ se ajusta sozinha e talvez não precise mexer; se continuar 0, IMPLEMENTAR o modelo por estoque. Nenhuma alteração de código feita ainda.

**Verificação agendada (cron LOCAL, não nuvem):** script `ecommerce-agent/scripts/check_havan_recebido_junho.py` agendado via crontab `30 7 12 6 *` (12/jun 07h30 local, após sync 06h). Lê o DB, checa `recebido_mensais['06/2026']` e manda Telegram: ✅ se populou / ⚠️ se ainda 0 (pedindo p/ implementar). É one-shot — remove a própria linha do crontab ao rodar. Log em `/home/ubuntu/havan_check.log`. NOTA: rota de nuvem (skill /schedule) foi descartada porque agente na nuvem não acessa o DB local de runtime.

Disparo manual do sync: `POST /api/havan/sync` (mesmo job das 06h). Token JWT mintável com `JWT_SECRET` default (não setado no .env). Ver [[ecommerce-havan-dashboard]], [[ecommerce-havan-backlog-faturamento-transito]].
