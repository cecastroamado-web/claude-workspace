---
name: ecommerce-nfe-items-sync-fix
description: "Bug crítico corrigido 05/jun/2026: sync de itens NF-e colapsava itens (item_id='') e job do scheduler DELETAVA itens validados de NFs com gap legítimo — lucro Havan inflado ~R$27k na NF 561. Lock nfe_items_lock + alerta Telegram de divergência."
metadata: 
  node_type: memory
  type: project
  originSessionId: 99287160-8075-4da9-b294-0e483951bee7
---

**Bug (corrigido 05/jun/2026, commit `8ef20ac` + scheduler em `fe41f68`):** lucro das vendas Havan inflado por itens de NF-e sumidos de `order_items_detail`.

Causa raiz dupla:
1. `bling.py sync_nfe_items_to_db` gravava todos os itens com `item_id=""` → UNIQUE(company, bling_order_id, ml_order_id, item_id) colapsava a NF inteira em 1 linha (entrou em `b5ad237`, 03/mai).
2. `scheduler._run_bling_nfe_items_sync` (6h) DELETAVA os itens de qualquer NF com gap > R$50 entre soma dos itens e valor_nota e re-sincronizava com o writer bugado (entrou em `ff7ae70`, 18/mai). NF 561 Havan tem gap LEGÍTIMO (devolução parcial: valor_nota 124.150 é o líquido; Bling traz 274.400/5 itens) → o job destruía os itens validados a cada restart. 29 NFs afetadas nas 2 empresas. Efeito P&L NF 561: CMV só do item devolvido, crédito ICMS subestimado → imposto inflado, lucro líquido 34.505 vs real 7.514.

**Why:** gap entre soma dos itens e valor_nota NÃO é sinônimo de erro de sync — pode ser devolução parcial, desconto na nota ou frete embutido (valorNota inclui frete em NF ao consumidor). Auto-"consertar" deletando é destrutivo.

**How to apply:**
- `nfe_items_lock` (bling_nfe_id, note): NFs com ajuste manual de itens — NUNCA re-sincronizar do Bling. NF 561 (24738765785) está travada; itens efetivos pós-devolução: 650 SLIM PRO@85 + 650 Cabo12v@56 + 650 USBC3m@50 = 124.150. Respeitado por: job do scheduler, `/api/sync-bling-nfe-revenue`, `/api/sync-nfe-items`.
- `db.replace_nfe_items`: substitui itens só DEPOIS do fetch Bling dar certo, preservando cost_override/icms_credit_rate_override por SKU.
- Gap persistente pós-sync (descontado o frete) > R$50 → alerta Telegram (estado `nfe_items_gap_alerted` em scheduler_state = {nfe_id: gap_sem_frete}; re-alerta só se gap mudar > R$1; NFs com gap conhecido são puladas no ciclo). Falhas de sync alertam 1×/dia (`nfe_items_sync_err_date`). Estado só persiste se `_notify` == True.
- Upsert de `bling_nfe` em sync-bling-nfe-revenue virou ON CONFLICT DO UPDATE — antes o REPLACE zerava custo_frete/vendedor/comissao/valor_devolvido/exclude_from_revenue.
- Backup com estado validado pré-bug: `data/ecommerce.db.bak_sku_migration` (19/abr).
- Gap que não fecha com frete → baixa o XML público da NF-e (`BlingClient.nfe_xml_totais`, link `xml` do GET /nfe/{id}) e desconta vSeg+vOutro+vIPI+vST−vDesc do ICMSTot antes de alertar (commit `96714e4`). Motivo: o JSON da API não expõe esses campos. NFs 50/51 Aviation tinham vOutro=79,90 (despesas acessórias) → gap 0,00, resolvidas.
- NFs com gap explicado (frete/despesas) ficam no estado `nfe_items_gap_alerted` e não re-sincronizam a cada ciclo.

Relacionado: [[ecommerce-havan-devolucoes]], [[nfe-discount-override-descontos-contratuais-por-nf]], [[ecommerce_havan_backlog_faturamento_transito]].
