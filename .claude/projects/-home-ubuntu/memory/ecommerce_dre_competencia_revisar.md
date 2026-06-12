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

**ACHADOS DA 1ª PASSADA (12/jun, DRE maio/2026 XConnect via `/api/dre`):** resultado op
R$ 61.343 (15,8%), líquido R$ 18.206. A DRE (`get_dre` em api.py:18814) monta:
receita − deduções(comissão+frete+DIFAL) − CMV(compute_profitability) − despesas_op(Google
Sheets + ADS) − tributos − abaixo da linha. **A dupla contagem está nas DESPESAS_OP do Sheets:**
`CATEGORIAS_EXCLUIR_DRE` (api.py:205) só remove INSUMOS/COMISSÕES/FRETE/ICMS, mas o Sheets traz
categorias que JÁ estão em outras linhas:
- **DIFAL** (Sheets R$ 2.630,61) ↔ DIFAL já nas deduções (R$ 10.478 de ml_nfe) → provável dobra.
- **IMPOSTOS** (Sheets R$ 3.814,42) ↔ tributos já calculados (PIS/COFINS/ICMS/IRPJ/CSLL R$ 51.570).
- **MARKETING** (Sheets R$ 31.424,89) ↔ ADS ML já somado à parte (R$ 15.047) → ver se o ADS está
  embutido no MARKETING do Sheets.
- **TARIFAS ENVIO FULL** (493,97) / **TARIFAS AFILIADOS** (693,74) / **DESPESAS COM VENDAS**
  (5.335,34) ↔ podem ser frete/comissão já nas deduções.
VALIDAR caso a caso olhando os lançamentos reais do Sheets (categoria pode ter nome igual mas
custo diferente — ex.: DIFAL de compra ≠ DIFAL de venda). Se confirmado, despesas inflam e o
resultado op/líquido fica SUBESTIMADO. Também auditar o CMV (compute_profitability não pode
embutir comissão/frete/imposto, senão dobra com as deduções) e a receita.

**AUDITORIA — STATUS (12/jun):**
- ✅ **CORRIGIDO (commit no submódulo): dupla contagem de ANÚNCIOS ML.** O ADS entrava 2× no DRE:
  `ads_ml` (ml_ad_spend, API, competência) + conta MARKETING do Sheets (favorecido "Mercado
  Livre", pagamento da fatura). Decisão CFO: manter `ads_ml`, excluir o do Sheets. Helper
  `_is_anuncio_ml_sheets()` (api.py) detecta por FAVORECIDO=Mercado Livre na conta MARKETING
  (robusto a obs vazia — Aviation lança sem 'ANUNCIOS'). **Impacto maio XConnect: resultado op
  61.343→91.918 (15,8%→23,7%); líquido 18.207→48.782.** Sistemático em jan/fev/mar/mai.
- ⏳ **IMPOSTOS no Sheets = imposto de IMPORTAÇÃO de imãs** ("600 IMÃS - AÉREO - FORNECEDOR
  CRISTIANO", R$ 3.814 mai / R$ 14.806 ano). NÃO é tributo de venda. É custo de insumo classificado
  como despesa op — ver se já está embutido no custo do imã (CMV) → dobra; se não, é custo legítimo
  mas mal-posicionado.
- ⏳ **DIFAL R$ 2.630 (Sheets, só maio, sem descrição)** — confirmar se é DIFAL de venda (dobra com
  as deduções de 10.478) ou de compra.
- ✅ **CORRIGIDO (padrão contábil, decisão CFO): "abaixo da linha".** Pró-labore → despesa op
  (entra no EBITDA); DIVIDENDOS e CAPEX NÃO reduzem mais o líquido (viram "Destinação do Lucro",
  informativo); `resultado_liquido = resultado_operacional`. Aplicado no cálculo principal,
  consolidado e CHART MENSAL (que também passou a excluir anúncios + incluir pró-labore). Front DRE
  reestruturado. **Impacto XConnect 2026: líquido −159.957 → +172.979** (o "prejuízo" era artefato
  de subtrair dividendos R$ 311k + pró-labore + capex). Maio: op 91.918→81.918 (pró-labore 10k
  agora é despesa), líquido = op.
- ⏳ Verificar se a dupla contagem de ADS existe em OUTRAS telas (overview/cashflow somam ml_ad_spend
  + despesas do Sheets?).

**FILA DE TAREFAS DO CFO (12/jun, NESTA ORDEM):**
1. **(atual) Auditoria DRE / resultado operacional** — confirmar e corrigir as duplas contagens.
2. **Cenários de pedido de imãs** — hoje o "pior caso" usa Pico de vendas ML + MAIOR mês Havan;
   como o melhor mês Havan tem pouco histórico, trocar a métrica Havan para **projeção de 30 dias
   pela velocidade de vendas de 7 dias** (consistente com o alerta/cobertura 7d). Ver
   [[ecommerce-ima-sugestao-pedido]].
3. **Projetar pedidos em ABERTO da Havan** no gráfico de Fluxo de Caixa e na Provisão de Caixa —
   desenhar como (timing de faturamento + entrega+120d). Ainda a conceber. Ver
   [[ecommerce-havan-backlog-faturamento-transito]] (já temos "a faturar" = pedidos_abertos −
   em_transito) e [[ecommerce-provisao-pontos-cegos]].

**O que revisar:** se a DRE do ecommerce reflete competência de VENDA ou de ENTRADA para
CMV/insumos/créditos de imposto, e se isso precisa ser alinhado ao regime contábil (ou exibido nas
duas óticas, como fizemos com a cobertura 7d × histórica no painel Havan). Confirmar com o CFO
**qual aba/relatório DRE** exatamente (o ecommerce-agent tem rentabilidade/overview/cashflow; uma
"DRE" formal pode ser nova ou estar em planilha).

Relacionado: [[ecommerce-rentabilidade-pedido]], modelo de ICMS (ecommerce_icms_model.md),
[[ecommerce-overview-categorias]].
