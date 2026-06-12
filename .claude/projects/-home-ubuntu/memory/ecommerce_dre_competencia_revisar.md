---
name: ecommerce-dre-competencia-revisar
description: Pendência (ecommerce-agent) — revisar a aba DRE e o tratamento de competência (venda vs entrada de insumo)
metadata: 
  node_type: memory
  type: project
  originSessionId: 5c7eb85e-fc5e-4fc1-80b9-ec8a21e0ed02
---

**PENDENTE (anotado 12/jun/2026, a pedido do CFO):** revisar a **aba/relatório DRE** e o
tratamento de **COMPETÊNCIA** no ecommerce-agent.

**⚠️ SUSPEITA FORTE DO CFO (12/jun) — AUDITORIA COMPLETA PEDIDA:** o **resultado operacional
reportado não parece fazer sentido** "olhando por cima" — parece estar **contando os MESMOS
custos em dobro**. Hipótese a investigar: o CMV/insumo entra duas vezes — ex.: a compra/reposição
de insumos (imãs, corpos injetados) lançada como SAÍDA no fluxo de caixa E o mesmo custo deduzido
de novo como CMV na rentabilidade/DRE; ou o custo do componente somado tanto na entrada quanto na
venda. Auditar TODA a cadeia de custo (CMV, componentes, frete, impostos, comissões) procurando
dupla contagem e competência trocada. Ver [[ecommerce-overview-categorias]] (identidade Variação
de Caixa = Δ saldo; baldes P&L vs financiamento vs investimento) — o risco de dupla contagem mora
exatamente na fronteira entre o balde de CMV (P&L) e o de reposição de estoque (caixa).

**Origem:** conferência do ICMS da matriz RS de maio/2026 (R$ 27.552 da contabilidade — confere).
Surgiu o ponto-chave de competência do CFO:
- O **painel de rentabilidade** (`compute_profitability` / `_calc_icms_net_month`) credita o ICMS
  dos insumos **na competência da VENDA** (custo do componente × alíquota, lançado no mês do
  pedido).
- A **apuração fiscal/contábil** credita **na competência da ENTRADA do insumo** (compra/nota de
  entrada), que costuma ser em meses anteriores (imãs/cases entram no estoque antes da venda).
- Por isso o crédito de ICMS do sistema (ótica venda) e o da contabilidade (ótica entrada) **não
  batem mês a mês** — ex.: maio RS, sistema estima crédito ~R$ 10,7k pela venda, fiscal real
  ~R$ 9,8k. O **débito** (ICMS das saídas) bate, é o **crédito** que diverge por timing.

**O que revisar:** se a DRE do ecommerce reflete competência de VENDA ou de ENTRADA para
CMV/insumos/créditos de imposto, e se isso precisa ser alinhado ao regime contábil (ou exibido nas
duas óticas, como fizemos com a cobertura 7d × histórica no painel Havan). Confirmar com o CFO
**qual aba/relatório DRE** exatamente (o ecommerce-agent tem rentabilidade/overview/cashflow; uma
"DRE" formal pode ser nova ou estar em planilha).

Relacionado: [[ecommerce-rentabilidade-pedido]], modelo de ICMS (ecommerce_icms_model.md),
[[ecommerce-overview-categorias]].
