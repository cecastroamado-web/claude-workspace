---
name: NFs Havan 547 e 552 — cancelamento/reemissão por regime (NÃO devolução)
description: NF Havan no bling_nfe sem entrada na planilha de recebíveis pode ser devolução OU cancelamento/reemissão por mudança de regime — verificar antes de assumir
type: project
originSessionId: c55d1021-f78f-437e-a9da-511f2b2ea0d7
---
⚠️ **CORREÇÃO (jun/2026, confirmado pelo usuário):** NF Havan **547 e 552 NÃO foram devolução**. Foram **canceladas em 2025 e reemitidas em 2026**, com o **produto de fato entregue em janeiro/2026**, porque a **Havan não compra de Simples Nacional** — a XConnect era Simples em 2025 e virou Lucro Presumido em 2026. Não tratar 547/552 como devolução de mercadoria, e não usá-las como "evidência histórica de devolução" em análises (foi premissa errada do card de sobre-estoque — ver [[ecommerce-havan-dashboard]]).

Implicação para dados: o "recebido" da Havan em jan/2026 (qtdRecebimento) é **entrega física real** (não é fantasma nem dupla contagem) — então o sinal de sobre-estoque/sell-through é **válido** (o estoque empilhou de verdade desde janeiro). O único erro era o rótulo "devolução".

Regra geral: NF Havan em `bling_nfe` com `situacao=6` sem entrada em "A RECEBER" pode ser (a) devolução real OU (b) cancelamento/reemissão por regime/timing. **Verificar antes de assumir.** O filtro de customer-ranking/profitability trata `valor_liquido = valor_nota - valor_devolvido` e descarta ≤ 0 (vale para devolução real registrada em `valor_devolvido`).

**Why:** O usuário já mencionou esse fato antes — não perguntar de novo se aparecer NF Havan no banco mas não na planilha de recebíveis.

**How to apply:** Antes de "investigar" NF Havan ausente do sheet, conferir `bling_nfe.valor_devolvido` ou perguntar de forma direta confirmando devolução, em vez de tratar como bug ou inconsistência.

**Neutralização aplicada (05/jun/2026):** 547/552 marcadas `exclude_from_revenue=1` no bling_nfe — no Bling seguem situacao=6 (autorizadas), mas comercialmente foram substituídas pela 561 (547+552 = R$ 274.400 = valor ORIGINAL da 561 pré-devolução). Sem a exclusão havia dupla contagem de R$ 274.400 de receita e ~R$ 82k de lucro no canal Havan. `/api/products` (top clientes) ganhou o filtro exclude no commit `3861970`. No mesmo dia: vendedor Rafael Ruczinski preenchido nas NFs 581/582/583 (Bling veio com vendedor id=0; padrão das outras 8 NFs Havan, comissão 15% auto) e mapa `nfe_product_cost_map` ganhou 'Cabo USBC 5 metros - #HVN#9000015140839#HVN#' → 'Cabo USBC 5 metros' (NF 581 tinha item sem_custo). Frete das 581/582/583 ainda zerado (não lançado).

**Retenção contratual 2% sobre valor ORIGINAL (mai/2026):** Contrato HAVAN prevê 1% comercial + 1% logístico sobre o valor original da nota, mesmo após devoluções parciais. Em NF com devolução, `valor_nota` (líquido pós-devolução) é menor que o original, então a regra padrão de `customer_discount_rules` (que aplica % sobre `valor_nota`) subestima o desconto. Caso NF 561: original R$ 274.400, devolução R$ 150.250, valor_nota R$ 124.150, retenção contratual R$ 5.488, líquido a receber R$ 118.662,00. Override gravado em `nfe_discount_override`. Ver [[nfe-discount-override-descontos-contratuais-por-nf]] para o mecanismo.
