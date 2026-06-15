---
name: ecommerce-havan-liquido-carry-sku
description: "ecommerce-agent — análise PLANEJADA (não implementada) do líquido Havan por SKU pós-carry de 121 dias, p/ definir piso de margem na renegociação"
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Líquido Havan por SKU pós-carry de 121 dias — análise planejada (CFO 15/jun/2026)

**Status:** DOCUMENTADO, nada codado. Aguardando o CFO fechar o XLSX de receita sem-nota antes de
implementar. Pedido do CFO para ter o **piso de margem real por SKU** na mão ao renegociar preço com a Havan.

## Contexto / decisão do CFO
- O canal B2B foi separado em **B2B avulso** vs **Havan** (ver [[ecommerce-dre-competencia-revisar]] — B2B
  2026 CMV 57,5% é a Havan, que é 80% da receita do canal; B2B avulso segue ~40% CMV).
- **Por que o B2B avulso tem mais margem:** produtos vendidos mais caros + **ativação embutida no produto**
  (serviço com cara de produto) → margem maior. A prateleira da Havan não comporta isso. São dois jogos
  diferentes — ok.
- **Avaliação do CFO sobre a Havan:** nos novos preços praticados, o **lucro líquido oscila de 20% a 32%**.
  O CFO **não acha ruim** para um atacadista grande de volume — e concordo (atacado costuma rodar líquido
  de 1 dígito a ~15%; 20-32% é forte).

## As 3 ressalvas (motivam a análise)
1. **Carry de 121 dias come o líquido.** A Havan paga em **121 dias** (ver [[ecommerce-havan-portal-pedidos]]).
   Selic ~15%/ano × 121/365 ≈ **~5 pontos percentuais** do faturamento parado. Um líquido contábil de 25%
   vira ~20% no econômico/caixa. Na ponta de baixo (20%) aperta. **O número que importa é o pós-carry.**
2. **Disciplina de piso de margem.** A 20% sobra pouco colchão p/ repique de custo (case China, câmbio USD,
   frete). Como o preço Havan é travado em contrato, um aumento de insumo joga p/ single digit sem avisar.
   Fixar um **piso de margem líquida pós-carry por SKU**.
3. **Concentração:** Havan = 80% do canal B2B (dependência de 1 comprador). Não é problema hoje, monitorar.

## Análise a implementar (líquido Havan pós-carry, por SKU)
Para cada SKU vendido à Havan, computar a margem líquida econômica:

```
líquido_sku = preço_atacado_Havan
            − CMV (custo versionado China 2026, via product_costs / nfe_product_cost_map)
            − comissão Mark (1% s/ bruto — ver [[ecommerce-havan-dashboard]])
            − impostos (XConnect Lucro Presumido: PIS/COFINS/ICMS/IRPJ/CSLL aplicáveis ao SKU)
            − carry de 121 dias  (≈ Selic_anual × 121/365 × preço_atacado)
```
- **Fontes:** `havan_nfe_items` / `havan_branch_product` (preço×qtd por SKU), `product_costs` (custo China
  2026 versionado), `nfe_product_cost_map` (mapa SKU→custo), Selic corrente (já lida do Sheets no IOF Sicredi).
- **Saída desejada:** tabela por SKU com `preço, CMV, %CMV, comissão, imposto, carry121d, líquido R$, líquido %`,
  ordenada por líquido % — destacando os SKUs abaixo do piso (cabos/suportes plásticos são os suspeitos:
  case c/ imãs ~39% margem produto, suporte SLIM ~29%, cabo USB-C ~64% CMV).
- **Uso:** munição para (a) renegociar preço nos SKUs abaixo do piso e (b) empurrar o mix p/ os melhores.
- **Caveat:** decidir se o carry entra como linha informativa (econômico) ou se vira custo financeiro de
  fato no líquido reportado — alinhar com o CFO. Não confundir com o resultado financeiro do DRE.

Relacionado: [[ecommerce-dre-competencia-revisar]], [[ecommerce-havan-dashboard]],
[[ecommerce-havan-portal-pedidos]], [[ecommerce-sicredi-emprestimo]] (Selic/spread).
