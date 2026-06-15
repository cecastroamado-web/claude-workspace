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

## Card "Havan a faturar" clicável + detalhe por OC (15/jun/2026)
O banner "🏬 Havan a faturar" na provisão (ProvisaoCard.tsx) virou **clicável** → abre Dialog com o
custo POR OC para entregar/faturar as OCs abertas. Endpoint **`GET /api/havan/aberto-detalhe`** (api.py)
reusa `_havan_oc_econ_mapper` + as taxas Havan: por OC devolve valor, ICMS líq (12%−créditos), PIS/COFINS
3,65%, IRPJ/CSLL 2,28%, imposto_total, cmv_avista (cabos/ventosas −30d) + cmv_corpo (corpos 50% +30/+60d),
cmv_total, custo_total, líquido + itens. Tipos `HavanAbertoDetalhe/OC/Item` + `api.havanAbertoDetalhe()`.
Validado 15/jun: 8 OCs, receita R$832.080 − insumos R$341.673 − impostos R$121.690 = líquido R$368.716.

## Simulação Havan — botão "Antecipar recebível da OC" (15/jun/2026)
Em `/simulacao-havan` (features/simulacao-havan/index.tsx): toggle "Calcular antecipação" + campos taxa
(default **1,8% a.m.**) e prazo (default **121d** = prazo de pgto da OC). Desconto **racional composto**:
VP = receita/(1+i)^(dias/30); custo = receita − VP; resultado antecipando = resultado − custo. Mostra VP
recebido hoje, custo da antecipação, resultado com vs sem. 1,8%/121d ≈ 6,9% da receita. **O PDF da
Simulação de Pedido inclui a seção "Antecipação do recebível"** quando ligada (HavanSimPdfBody ganhou
antecipar/antecip_taxa_am/antecip_dias; `build_havan_simulacao_pdf` renderiza KPIs+tabela).

## Prazo de pagamento dos insumos das OCs Havan (decisão CFO 15/jun/2026)
`_havan_oc_produto_econ` passou a separar **cabo / ventosa / corpo** (antes cabo+ventosa juntos em
"avista"). Prazos em `_havan_aberto_cmv_por_mes` (compra ~30d antes da entrega = `pd`):
- **Cabos**: 100% à vista (`pd`).
- **Ventosas**: 50% à vista (`pd`) + saldo 50% em 30/60d da compra (25% `pd+30`, 25% `pd+60`).
- **Bases dos cases (fornecedor Gedeval = bucket `corpo`, + demais produzidos)**: **120 dias do
  faturamento ≈ entrega → `entrega+120d`** (alinha com o recebível Havan de ~121d). Antes era 50% +30 / 50% +60.
Total do CMV não muda (R$ 341.673 nas OCs atuais), só a distribuição mensal (bases pulam p/ out–dez).
**Atualização 15/jun (CFO): ventosas são COMPRA EM BLOCO ÚNICO** (não dá p/ quebrar; comprar já p/
preparar). Constante `_HAVAN_VENTOSA_COMPRA_DATA="2026-06-19"`: 50% em 19/06 + 25%/25% em 30/60d da
compra, consolidando TODAS as OCs num só lançamento (pedido="(todas as OCs)"). Cabos e bases seguem
por OC. Validado: ventosas R$ 31.720 (10.400×3,05) → 15.860 (19/06) + 7.930 (19/07) + 7.930 (18/08).
Custo unitário R$ 3,05 e qtd (Slim=4, Painel=2 → 10.400) confirmados pelo CFO; mantém 10.400 sem abater
o pouco em estoque. Bases (corpo) R$ 146.828 confirmadas (vêm de product_costs, exceto imã/ventosa).
Textos do modal/PDF atualizados; detalhe expõe cmv_avista (= cabos+ventosas, rotulado "acessórios") e
cmv_corpo ("bases"). Imposto inalterado (usa só o crédito de ICMS).

## Fluxo de caixa da operação no tempo (mês a mês) no detalhe (15/jun/2026)
`/api/havan/aberto-detalhe` ganhou **`fluxo_mensal`**: por mês, recebimento Havan (entrada, no
data_recebimento_est), insumos (saída, mesmo `_havan_aberto_cmv_por_mes`), impostos (saída,
`_havan_aberto_impostos_por_mes`), líquido do mês e **acumulado** (caixa imobilizado até a Havan pagar).
Mostra o desencaixe REAL no tempo, não só o líquido por OC. Modal (ProvisaoCard) tem tabela "📅 Desencaixe
da operação — mês a mês"; PDF tem seção idem. Validado 15/jun: fundo −R$316.535 em set/26, fecha
+R$368.716 (= líquido total) em dez/26 com os recebíveis.

## Card detalhe maior + PDF do desencaixe + antecipação das OCs no FLUXO (15/jun/2026)
- Modal "Havan a faturar" (ProvisaoCard) expandido p/ ~96vw×92vh + **botão "📄 PDF do desencaixe"**
  (`GET /api/havan/aberto-detalhe/pdf` → `build_havan_aberto_pdf` no havan_report.py; KPIs receita/insumos/
  impostos/líquido + tabela por OC). `api.havanAbertoDetalhePdf()` baixa blob.
- **Antecipar OCs a faturar NO FLUXO** (`/api/cashflow` param **`antecipa_havan_aberto`**, reusa
  taxa_am/modelo): cada OC entra LÍQUIDA no mês da ENTREGA (ou hoje, se já entregue) em vez do bruto em
  entrega+121d; desconto = custo. Resumo `antecipa_havan_aberto` (qtd/bruto/desconto/líquido/títulos).
  ⚠️ Diferente do `antecipa_havan` existente, que antecipa as FATURAS JÁ EMITIDAS (status A RECEBER) p/ hoje.
  Toggle próprio "Antecipar OCs Havan a faturar (na entrega)" no cashflow, vale nos 2 modos. Validado
  15/jun: 8 OCs, bruto R$832.080, desconto R$57.768,50 (6,94%, 121d), líquido R$774.311,50.
