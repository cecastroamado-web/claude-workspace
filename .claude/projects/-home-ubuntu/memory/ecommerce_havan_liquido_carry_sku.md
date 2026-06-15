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

## ✅ EXECUTADO (15/jun/2026) — resultado + verificações
**Líquido por SKU (cenário forward, pós-carry 121d) — premissas: Selic 1,07%/mês (carry 4,32%), ICMS déb 9,3% blended real − crédito, PIS/COFINS 3,65%, IRPJ/CSLL presumido 3,08%, comissão Mark 1%:**
- Slim Pro 2 (corpo 25,94 + 4 ventosas transp 3,05 = R$38,14): líq **39,6%**
- Haste (48,62): **36,2%** · Cabo 12v 3m (25 flat): **34,0%** · Cabo USB-C 5m: **34,0%** · Cabo 12v 2m: **32,4%** · Cabo USB-C 3m: **28,7%**
- Case Plástico c/ Imãs **88mm** (corpo 49,25 + imã 88mm 127,04 = R$176,29): líq **23,5%** ← menor margem, maior receita; é o piso real
- **PONDERADO LIMPO: 30,6%** (receita R$966.170 = Havan 2026; líquido R$296.002). Excel `havan_liquido_por_sku.xlsx` no Telegram (3 abas).

**Verificações decisivas (corrigem suposições anteriores):**
1. **Custo do imã 88mm NÃO está no `product_costs`** — está em **`order_item_component_override`** (component 121 → R$127,04) POR PEDIDO. O `compute_profitability` aplica via `_load_item_component_overrides`. Por isso meu 1º diagnóstico ("DRE subcusteia") estava ERRADO — eu só tinha olhado product_costs.
2. **DRE custeia os cases corretamente:** NF 552 (dez/2025) = 66mm (tinha estoque); de jan/2026 (NF 564) em diante = 88mm. A virada bate com o esgotamento do 66mm. Override cobre 564/579/582; 552 fica 66mm e está CERTO.
3. **NFs canceladas NÃO entram no DRE:** 547 e 552 têm `exclude_from_revenue=1` + `valor_devolvido`=cheio. A query do DRE (`api.py:6952`, `situacao=6 AND exclude_from_revenue=0`) as tira da receita E do CMV (não vão pro orders_fees → compute_profitability nem processa). Sem custo-fantasma. **Regra geral: cancelada nunca entra no DRE.**
4. Quantidades LIMPAS (order_items_detail, mapeado por valor à NF, excluindo 547/552): case 1.300 (−300 da 552), haste 1.150 (−650), slim 2.150 (−650), cabo 12v 3m 2.720 (−650), cabo USB-C 3m 1.555 (−650). Cabos 2m/USB-C 5m sem mudança.

**Pendências reais (não-case):** (a) Slim Pro 2 e cabos R$25 são mudança de produto/fornecedor — registrar no `product_costs` (forward) só quando confirmado p/ todos os canais; cabos R$25 ainda é **custo-alvo** (fornecedor regime normal pendente → +4% crédito ICMS depois). (b) NF 561 (devolução travada) — verificar à parte se a parte não-devolvida tem case a contar.
