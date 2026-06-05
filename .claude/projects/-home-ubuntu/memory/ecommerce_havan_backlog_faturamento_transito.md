---
name: ecommerce_havan_backlog_faturamento_transito
description: "Em-trânsito Havan: agregado por mês (6127d17) + POR PRODUTO em itens (7ad4de9, via havan_nfe_items + regra de data). Caveat: snapshot recebido SUBCONTA"
metadata: 
  node_type: memory
  type: project
  originSessionId: 04aa6148-ffed-4da9-bc7c-97d934b195f6
---

# Faturamento Havan de produtos em trânsito — RESOLVIDO (04/jun/2026)

**Resolução:** commit `6127d17` "feat(havan): #5 conciliação mês a mês + NF não-entrada"
entregou a separação **agregada por mês/valor**: KPI "Em trânsito" = `faturado − recebido`,
tabela mensal com coluna "Não-entrada", e o detalhe de que o pagamento é **entrega + 120d**
(o relógio dos 120d só começa quando a Havan registra recebimento). Atualiza-se sozinho
conforme a Havan recebe. CFO decidiu que **o agregado já basta** — não fazer camada por-NF.

**Origem:** NFs Havan de 01/06/2026 (~R$ 286k) emitidas com mercadoria ainda em trânsito,
aguardando agendamento da Havan, entrega provável **09–10/jun/2026**. Achado do `6127d17`:
jun/2026 R$286k faturado, R$0 recebido → ~R$288k em trânsito.

**Correção de dado:** as NFs de junho em trânsito são **SLIM PRO / HASTE / cabos 12V**
(NF 581/582/583), NÃO cases. O "~970 cases" que eu havia estimado por valor÷R$295 estava
errado (a NF mistura produtos).

## Em-trânsito POR PRODUTO em itens (05/jun/2026, commit `7ad4de9`)
Coluna "Em trânsito" (itens) na tabela por-produto da Conciliação + KPI "N itens em NF não
entregue". Valores validados vs gabarito do CFO: **Cabo 12V 3m 1.000, Slim 850, Case 260,
USB-C 5m 600, Haste 500 = 3.210 un** (todas das NFs de 01/06).

**Definição correta (trava a fórmula):** em-trânsito = itens das **NF-e ainda não entregues
no CD** = `data_emissao + ~15d > hoje` (`_HAVAN_TRANSITO_DELAY_DAYS`). Tudo emitido antes
disso a Havan já recebeu. Regra dada pelo CFO: "a única coisa não entregue é a nota de 01/06".

**⚠️ CAVEAT CRÍTICO — NÃO usar `faturado − recebido` do snapshot:** o campo `recebido_periodo`/
`recebido_mensais` do snapshot Havan **SUBCONTA** o que a Havan recebeu (ex.: dizia 650 hastes
quando o real é 1.800). `faturado − recebido_snapshot` infla o trânsito. A medida confiável é
**itens das NFs não entregues** (regra de data acima).

**Fonte dos itens:** tabela dedicada **`havan_nfe_items`** (nfe_id, sku, qty, data_emissao;
PK nfe+sku; sync lazy via `get_nfe_details` agrupando por código). O bug antigo do
`_run_bling_nfe_items_sync` (itens colapsados/destruídos em `order_items_detail`) foi
**CORRIGIDO em 05/jun/2026** — ver [[ecommerce-nfe-items-sync-fix]]. `order_items_detail`
voltou a ser confiável p/ rentabilidade; `havan_nfe_items` segue sendo a fonte do trânsito.
Atenção: `havan_nfe_items` da NF 561 tem os 5 SKUs FATURADOS (inclui os devolvidos) —
correto p/ trânsito histórico, mas não usar p/ receita.

**SKU truncado na NF:** o código na NF pode vir cortado (`HVNCSPLIM660` na NF vs
`HVNCSPLIM66007V` no snapshot) → match por igualdade e, no resto, por prefixo (`_havan_match_faturado`).

**Coluna "A faturar"** (commit `ce88c49`, sessão paralela) = `pedidos_abertos − em_trânsito`
(o que a Havan pediu e ainda não enviamos). Em 05/jun/2026 (`ae14572`) a mesma regra foi
aplicada ao **card KPI "Backlog a faturar"** (ex-"Backlog em aberto") — valor e unidades
descontam o em-trânsito; ver [[ecommerce-havan-dashboard]].

**Plano arquivado** (camada por-NF com status/confirmação manual/matching #HVN#, caso um dia
queiram auditoria por nota): `/home/ubuntu/.claude/plans/majestic-stirring-willow.md`.
Relacionado: [[ecommerce_ima_sugestao_pedido]].
