---
name: ecommerce-ventosa-utimex-credito-simples
description: "Ventosa UTIMEX (Simples) concede crédito de ICMS 3,89% na NF — Simples nem sempre é 0% de crédito"
metadata: 
  node_type: memory
  type: project
  originSessionId: 85b1fdaa-2bbf-46c3-bef8-79b101b760fb
---

NF de entrada 33.929 (emissão 23/06/2026), fornecedor **UTIMEX Ind. e Com. de Plásticos** (CNPJ 33.531.439/0001-08, SP, **Simples Nacional**), destinatário **XConnect matriz RS** (57.710.730/0001-01): VENTOSA 65mm c/ parafuso/porca M6 (NCM 40021919), **9.000 un × R$ 3,05 = R$ 27.450,00**, pgto 50/25/25 (23/06, 23/07, 22/08).

**Premissa corrigida:** fornecedor Simples Nacional **nem sempre** dá 0% de crédito de ICMS. Quando a NF **informa** o crédito (art. 23 LC 123), o comprador no Lucro Presumido **pode aproveitar**. Esta NF concede **R$ 1.067,81 = 3,89%**. Difere dos cabos Simples (ver [[ecommerce-rentabilidade-padronizacao]]), onde o fornecedor NÃO informou crédito → ficou 0%. **Regra: checar o campo "Inf. Contribuinte / Permite aproveitamento de crédito" na NF de entrada Simples antes de assumir 0%.**

**Uso (decidido):** a ventosa transparente entra no **Slim Pro 2** (4 un × 3,05 = R$ 12,20), que **substitui** o Slim Pro antigo (mas pode voltar no futuro — substituição temporal, não simultânea). Slim Pro 2 **ganha SKU NOVO no Bling** (ainda desconhecido — "saberemos assim que for faturado"), então é **produto separado**, NÃO versionamento do antigo.

Slim Pro **antigo** (`Case Slim Pro para vidros retos`) fica **intacto**: 3× ventosa de **sucção** (3 × 11,57 = R$ 34,71, rate 0.04). Minha 1ª inserção (12,20 com effective_from=23/06 nesse produto) foi **REVERTIDA**.

**Gravado** novo produto **`Case Slim Pro 2`** (XConnect + Aviation), `effective_from='2026-06-23'`: Produto/base id=1 = R$ 25,935 @ 0.17 (**CONFIRMADO pelo CFO** — igual ao corpo do antigo); Ventosas id=122 = R$ 12,20 @ **0.0389** (XConnect LP) / **0.0** (Aviation Simples). CMV total R$ 38,135.

**PENDENTE (quando Slim Pro 2 for faturado):** pegar o SKU novo + nome real da NF e (1) setar `sku` nas linhas de `product_costs` do `Case Slim Pro 2` e (2) adicionar alias em `nfe_product_cost_map` (nome NF → `Case Slim Pro 2`), senão as vendas não casam o custo. Análise de rentabilidade que fiz antes (Havan @R$85: margem 16,5%→41,9%) só vale DEPOIS que a Havan migrar pro Slim Pro 2 e o alias existir; hoje a Havan ainda vende o antigo (sucção 34,71).

Caveats: a UNIQUE(company,product_name,component_id,effective_from) de `product_costs` NÃO está realmente aplicada neste DB (há linhas duplicadas 2000-01-01) → `ON CONFLICT` falha; inserir nova data com INSERT puro. Aviation é Simples → não credita ICMS de qualquer forma (não lançado lá). Ver [[ecommerce-icms-aliquotas-reais]].
