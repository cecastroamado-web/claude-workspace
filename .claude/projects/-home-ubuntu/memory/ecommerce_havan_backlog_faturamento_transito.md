---
name: ecommerce_havan_backlog_faturamento_transito
description: "RESOLVIDO (6127d17) — em-trânsito Havan agregado por mês; sobra acompanhar entrega real 09-10/jun"
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
errado (a NF mistura produtos). A quantidade real por produto sai item-level de
`order_items_detail` (join `bling_order_id = bling_nfe_id`), com o código Havan embutido
no nome do item como `#HVN#<produtoCodigo>#HVN#`.

**Único acompanhamento que sobra (operacional):** confirmar a entrega real após 09-10/jun
(o em-trânsito deve cair sozinho quando a Havan registrar o recebimento).

**Plano arquivado** (camada por-NF com status/confirmação manual/matching #HVN#, caso um dia
queiram auditoria por nota): `/home/ubuntu/.claude/plans/majestic-stirring-willow.md`.
Relacionado: [[ecommerce_ima_sugestao_pedido]].
