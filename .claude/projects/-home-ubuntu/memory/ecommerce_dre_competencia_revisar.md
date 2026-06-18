---
name: ecommerce-dre-competencia-revisar
description: Pendência (ecommerce-agent) — revisar a aba DRE e o tratamento de competência (venda vs entrada de insumo)
metadata: 
  node_type: memory
  type: project
  originSessionId: 5c7eb85e-fc5e-4fc1-80b9-ec8a21e0ed02
---

**✅ Pró-labore corrigido (18/jun):** `_classifica` devolve "abaixo_linha" p/ pró-labore, e os endpoints `/api/overview/competencia` + o de detalhe/categoria (api.py ~1921 e ~2540) **dropavam** ele (não entrava em despesa_op nem em abaixo_linha=div+capex) → resultado operacional superestimado em TODOS os períodos (XConnect: 2024 +41k, 2025 +240k, 2026 +230k; acum. 511k). Fix localizado: somar pró-labore no despesa_op desses 2 (constante `_PROLABORE_NORM`) + na quebra by_category. Demais consumidores (cash overview 2039, DRE endpoint 21080, cashflow) já o tratavam como despesa → não tocados (sem dupla contagem). Commit `e515bac`.

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
- ✅ **AUDITADO 13/jun — IMPOSTOS no Sheets NÃO dobra com o CMV (premissa da memória era errada).**
  São 6 lançamentos XConnect (total R$ 22.892; 2026 = R$ 14.806), e a MAIORIA é imposto sobre
  **remessa de EXPORTAÇÃO** (Argentina/Colômbia), não importação de imã:
  - #1-4 (R$ 8.549, 2025-26): impostos de remessa export p/ ARG/COL (reemb. ao Carlos) — custo real.
  - #5 (R$ 10.527, mar/26, PORTIQ despachante Colômbia): **CFO confirmou = despesa real do XConnect**
    (apesar da obs "Colômbia pagou→Carlos→CR2"). Fica no DRE.
  - #6 (R$ 3.814, mai/26, FedEx, "600 IMÃS AÉREO FORNECEDOR CRISTIANO"): único de importação de imã.
    **CFO confirmou que os 600 imãs aéreos são custeados FOB (sem imposto)** → o imposto só aparece
    na aba IMPOSTOS, NÃO dobra com o CMV. (CMV do 66mm China = R$ 16,23/un na `ima_prices` é landed
    do lote MARÍTIMO — id=2, "1º pedido China Marítimo"; o lote aéreo Cristiano é separado.)
  **Conclusão: nenhuma mudança de código.** Todos os 6 são custo legítimo e devem permanecer em
  despesas_op. Caveat cosmético: #6 é custo de insumo lançado como despesa op (devia ser CMV), mas
  como o imã é FOB, o total do resultado está correto — é só questão de LINHA, não de valor.
- ✅ **CORRIGIDO (commit `16bf10e`): DIFAL R$ 2.630 do Sheets é DIFAL de VENDA recolhido pelo ML.**
  Validado: única linha DIFAL em 5.505 do Sheets tem FAVORECIDO=Mercado Livre, competência
  maio/2026 (conta INTER X CONNECT). É o mesmo DIFAL já nas deduções (`valor_icms_ufdest` de
  ml_nfe) — estava sendo somado em despesas_op, dobrando. `_is_anuncio_ml_sheets` generalizado
  p/ `_is_ml_ja_capturado_sheets`: exclui de despesas_op os lançamentos pagos ao ML nas contas
  MARKETING (anúncios, já em ads_ml) E DIFAL (já nas deduções). Não há DIFAL de compra com outro
  favorecido, então a exclusão é segura.
- ✅ **CORRIGIDO (padrão contábil, decisão CFO): "abaixo da linha".** Pró-labore → despesa op
  (entra no EBITDA); DIVIDENDOS e CAPEX NÃO reduzem mais o líquido (viram "Destinação do Lucro",
  informativo); `resultado_liquido = resultado_operacional`. Aplicado no cálculo principal,
  consolidado e CHART MENSAL (que também passou a excluir anúncios + incluir pró-labore). Front DRE
  reestruturado. **Impacto XConnect 2026: líquido −159.957 → +172.979** (o "prejuízo" era artefato
  de subtrair dividendos R$ 311k + pró-labore + capex). Maio: op 91.918→81.918 (pró-labore 10k
  agora é despesa), líquido = op.
- ✅ **VERIFICADO 13/jun — SEM dupla contagem de ADS em overview nem cashflow.** Overview
  (`get_overview`) NÃO usa `ml_ad_spend` — só Sheets via `_filter_sheets_cash` → ADS uma vez.
  Cashflow (`get_cashflow`): histórico usa só Sheets (sem ml_ad_spend); projeção futura usa só
  `_ads_by_month` (API, via `_ads_invoice_schedule`) e `a_pagar_op` NÃO recebe ADS — os 29
  lançamentos MARKETING+Mercado Livre estão TODOS `OK`/efetivados (zero pendentes A VENCER/VENCIDO,
  e a_pagar_op só pega pendentes); `ads_est` ainda exclui faturas vencidas ≤hoje (já no saldo).
  Nenhuma mudança de código. (Um agente de busca deu falso positivo de "crítico" — refutado por
  checagem dos dados reais.) **Risco latente** (não corrigido): se o CFO lançar fatura FUTURA de
  anúncio ML como "A VENCER", entraria em a_pagar_op + _ads_by_month → dobraria. Guarda defensiva
  opcional (excluir ML ADS de a_pagar_op, como o DRE) — proposta ao CFO, sem decisão ainda.
- ✅ **CORRIGIDO 13/jun (commit `83f4030`): ADS de períodos anteriores via Sheets quando a API não
  cobre.** A API do ML não traz histórico antigo (`ml_ad_spend`: XConnect ≥ 2025-12, Aviation ≥
  2026-01; contígua, sem buracos no meio). Buraco real: **XConnect 2025-03..2025-11 (9 meses) =
  ~R$ 147.839 de ADS que sumiam do DRE**. Fix: `_is_ml_ja_capturado_sheets` refatorado em
  `_is_difal_ml_sheets` (DIFAL ML — sempre exclui) + `_is_anuncio_ml_sheets` (MARKETING ML — exclui
  SÓ se o mês está coberto pela API) + helper `_row_year_month`. No `get_dre`: cálculo principal usa
  `_ads_api_months` (set de YYYY-MM por empresa); chart mensal usa `_ads_api_cov` (set
  (empresa,YYYY-MM)). Meses cobertos → API; anteriores → Sheets. Validado: XConnect 2025 ADS
  3.108→~150.947; 2026 inalterado (API-only, sem dobra).
  ⚠️ **Caveat boundary**: o 1º mês coberto (XConnect 2025-12) tem API parcial (R$ 3.108 vs Sheets
  R$ 20.311 — sync inicial incompleto). Como 2025-12 está no set da API, usa o valor da API (baixo).
  Avaliar se vale tratar a fronteira (ex.: usar o maior entre API e Sheets no 1º mês). Não crítico.

**FILA DE TAREFAS DO CFO (12/jun, NESTA ORDEM):**
1. **(atual) Auditoria DRE / resultado operacional** — confirmar e corrigir as duplas contagens.
2. ✅ **FEITO 13/jun (commit `79252a1`)** — a SUGESTÃO primária de pedido de imãs passou a usar o
   **Ritmo 7d** (ML+Havan venda_7d×30/7); o pico (ML pico + Havan maior mês) virou stress separado
   (`*_pico`). Box "Ritmo 7d" = ★ recomendado. Ver [[ecommerce-ima-sugestao-pedido]].
3. ✅ **FEITO 13/jun — pedidos abertos da Havan no Fluxo de Caixa (`148ba73`) E na Provisão
   (`74b7dea`).** Fluxo: linha "Havan a faturar" (recebimento = entrega + 121d). Provisão: os que
   caem no horizonte (≤60d) entram no day-by-day + resumo informativo `havan_aberto` {total, qtd,
   por_mes} (os R$ 832k caem Out-Dez, além dos 60d → banner no ProvisaoCard). Ver
   [[ecommerce-havan-portal-pedidos]]. Contexto histórico:
   desenhar como (timing de faturamento + entrega+120d). Ainda a conceber. Ver
   [[ecommerce-havan-backlog-faturamento-transito]] (já temos "a faturar" = pedidos_abertos −
   em_transito) e [[ecommerce-provisao-pontos-cegos]].
   - **Reforço CFO (12-13/jun):** especificamente os **pedidos AINDA NÃO FATURADOS** da Havan
     precisam entrar no **Fluxo de Caixa** para dar **visão de QUANDO vão ser recebidos** (entrada
     de caixa futura). Hoje o fluxo só enxerga o que já virou NF/recebível. Modelar a régua:
     pedido aberto → data estimada de faturamento → recebimento (prazo Havan, ~entrega+120d). É o
     elo que falta para o fluxo refletir o pipeline comercial Havan, não só o faturado.

**O que revisar:** se a DRE do ecommerce reflete competência de VENDA ou de ENTRADA para
CMV/insumos/créditos de imposto, e se isso precisa ser alinhado ao regime contábil (ou exibido nas
duas óticas, como fizemos com a cobertura 7d × histórica no painel Havan). Confirmar com o CFO
**qual aba/relatório DRE** exatamente (o ecommerce-agent tem rentabilidade/overview/cashflow; uma
"DRE" formal pode ser nova ou estar em planilha).

## 📋 VARREDURA AUTÔNOMA COMPLETA 13/jun (CFO volta 14/jun p/ validar JUNTOS) — checklist
Revisão de TODAS as possibilidades do DRE 2026 p/ não deixar margem de erro. Status de cada item:

**🔴 BUGS ADICIONAIS achados na auditoria autônoma da madrugada (14/jun) — A VALIDAR:**
A1. **[CRÍTICO] DIFAL DOBRADO no DRE ANUAL XConnect** (subagente fiscal). DRE anual mostra DIFAL
   R$ 231.298 = **2× o real R$ 116.053** (fonte `ml_nfe` Σ valor_icms_ufdest+fcp, n=3.676; bate ao
   centavo: 116.053 + 115.244 do _calc_icms_net = 231.298). Causa: `difal_val` é pré-carregado com o
   SQL bruto de ml_nfe (`api.py` ~19528-19538, fora do branch de regime) e o **path ANUAL** (linha
   ~19815 `difal_val += _i["difal"]`) acumula EM CIMA do seed → dobra. Mensal e consolidado-mensal OK;
   **anual e consolidado-anual** dobram. DIFAL entra nas DEDUÇÕES → infla deduções → **subestima
   resultado op 2026 XConnect em ~R$ 115k**. (No `tributos.total` o DIFAL não entra — isso está certo.)
   **Fix:** zerar `difal_val = 0.0` antes do loop anual no branch LP (junto de icms_debito/credito/liq),
   ou não somar sobre o seed. ⚠️ Confirma a suspeita do CFO de que R$ 231k estava alto.
A2. **[ALTA] CMV diverge entre Rentabilidade e DRE em R$ 62.372** (subagente consistência). O DRE chama
   `compute_profitability` SEM passar `product_components_history` + `item_component_overrides` +
   `icms_credit_rate_history` (api.py ~19741), então usa custo de componente padrão e **subestima o CMV**
   de 3 NF-e HAVAN (NF 582 +16.151, NF 579 +45.969, NF 564 +252). `/api/profitability` passa esses args
   (CMV mais alto). → DRE **superestima o lucro bruto XConnect em R$ 62k**; as duas telas mostram CMV
   diferente p/ os mesmos pedidos. Fix: passar os mesmos args no DRE.
A3. **[ALTA] `/api/profitability` (painel Rentabilidade) IGNORA NFS-e** (receita E lucro). Aviation:
   receita Rentab 688.540 vs DRE 740.360 → diferença = **R$ 51.820 = a NFS-e**. O painel só tem fontes
   ml_direct + bling_manual, sem `nfse`. **É a origem do 690k(CFO) × 644k(painel atual):** 644.173 +
   47.705 (lucro NFS-e omitido) = 691.878 ≈ 690k. Ou seja, o **R$ 690.089/21,3% que o CFO citou é a
   Rentabilidade COM NFS-e**; o painel hoje mostra 644k/20,17% por excluir NFS-e. Fix: incluir NFS-e no
   /api/profitability (receita+lucro). ⚠️ a margem do painel também usa denominador sem NFS-e.
A4. **[MÉDIA] Comissão de vendedor (auto) só na Rentabilidade, NÃO no DRE.** `/api/profitability` deduz
   auto-vendor-commission (XC 29.350 + AV 14.067 = R$ 43.417); o DRE não aplica (só usa a coluna `cmv`
   do profit_df) E exclui COMISSÕES do Sheets → a comissão de vendedor **não aparece em lugar nenhum do
   DRE** → DRE superestima resultado em ~R$ 43k (ou ~R$ 12k se considerar só o lançado em COMISSÕES no
   Sheets). Decidir: manter COMISSÕES no DRE OU aplicar a comissão no compute_profitability do DRE.
A5. **[BAIXA] Chart mensal do DRE: NFS-e sem filtro `exclude_from_revenue`** (api.py ~20090) — total
   anual OK; só as barras mensais podem superestimar se houver NFS-e excluída (hoje impacto R$ 0).
A6. **[BAIXA] Reconciliação XConnect bate bem** (após fix do "todas"): profitability lucro 423.065 −
   overhead 256.905 = 166.160 ≈ DRE resultado op 165.671 (gap R$ 489, arredondamento). Modelo coerente.

**🔴 BUGS (fix pronto, NÃO commitado — validar e aplicar):**
1. **DRE "todas" dobra a despesa** (`api.py:19844` `empresa_sheets = company if not todas else None`).
   No consolidado, cada empresa processa TODAS as despesas do Sheets → despesa 2× + Aviation
   negativo. **Fix: `empresa_sheets = company`.** Impacto: XC 153.900→**165.698**; AV −7.751→
   **+249.289**; todas 146.288→**414.986** (= soma real). Validado (filtra certo com nome capitalizado).
2. **FRETE — excluir SÓ `SUB-DESCRIÇÃO=HAVAN`** (não tudo). FRETE hoje 100% em CATEGORIAS_EXCLUIR_DRE.
   Split 2026: HAVAN R$ 18.400 (entrega Havan, JÁ na rentabilidade por NF → EXCLUIR); FULL+inbound
   R$ 44.496 XC + R$ 15.320 AV (Expresso Lajeado/São Miguel/DHL → MANTER como despesa op). Fix: tirar
   FRETE do EXCLUIR + regra que pula linha com SUB-DESCRIÇÃO=HAVAN. Direção: +despesa, −resultado.

**🟢 VERIFICADOS CORRETOS (sem ação):**
3. COMERCIAL Rafael R$ 42.000 = **salário fixo R$ 7.000/mês recorrente** (CFO confirmou) → mantido no
   DRE (overhead, NÃO está na rentabilidade). ✓
4. COMISSÕES (Rafael R$ 5.262 + Mark Barbouth R$ 6.801) = comissão de venda → **já excluída**
   (CATEGORIAS_EXCLUIR_DRE) e ambas na rentabilidade (Rafael 15% s/ lucro, Mark 1% s/ bruto). ✓
5. INSUMOS R$ 1,98M (compras de insumo, com SUB IMAS/CASES/CABOS/ANTENAS/VENTOSAS) → excluída; o DRE
   usa CMV de `compute_profitability` (R$ 1,36M, base SOLD/competência). Correto (compra ≠ CMV vendido).
6. DIFAL R$ 231.298 aparece na linha de tributos MAS **não é somado ao total** (já nas deduções) →
   sem dobra. Tributos total 307.249 = ICMS líq 158.684 + PIS 16.284 + COFINS 75.159 + IRPJ 30.063 +
   CSLL 27.057. (⚠️ DIFAL R$ 231k é ALTO — confirmar com CFO/contador se faz sentido.)
7. Categoria F mapeada 100% (XC+AV) — nenhuma órfã. ADS já corrigido (commits anteriores).

**🟢 RESOLVIDO (CFO confirmou):**
8. **DESPESAS COM VENDAS R$ 24.442 (XC) NÃO está na rentabilidade** → overhead legítimo, **manter
   TUDO no DRE** (incl. etiquetas Havan R$ 525, Correios R$ 3.844, reembolsos/viagens Rafael,
   consultoria E/PURO, Shopify). Sem dobra (a rentabilidade só deduz comissão/CMV/frete venda/ICMS/
   DIFAL/imp.federais/ADS — não "despesas com vendas"). ✓
9. Outras categorias com SUB-DESCRIÇÃO=HAVAN além de FRETE: só DESPESAS COM VENDAS (que fica no DRE
   inteiro — ver item 8). Resto dos SUB são insumos (IMAS/CASES/CABOS/ANTENAS) já via INSUMOS/CMV.

**🔵 RECONCILIAÇÃO (validar a lógica):** Rentabilidade 2026 R$ 690.089 (21,3%) = margem de
contribuição (deduz comissão ML, CMV, comissão vendedor, frete venda, ICMS, DIFAL, imp. federais,
ADS). DRE = rentabilidade − OVERHEAD (salários/pró-labore/aluguel/contador/ERP/internet/energia/
prev./legais/reembolso/frete inbound-Full). DRE consolidado CORRIGIDO ≈ R$ 415k (antes do fix do
FRETE) → ~R$ 355k depois de incluir o frete inbound (~R$ 60k). Gap p/ a rentabilidade ≈ overhead +
diferença de método nos tributos (DRE estima por alíquota × receita vs rentabilidade usa o real).
**A fechar na validação:** bater o overhead linha a linha e a diferença de tributos estimado×real.

## 🎯 BUG ENCONTRADO 13/jun (autônomo, A VALIDAR/aplicar) — DRE "todas" conta despesa em DOBRO
**Causa raiz dos números estranhos.** `api.py:19844` (dentro de `get_dre`, loop `for company in
target_companies`): `empresa_sheets = company if not todas else None`. No modo **"todas"**,
`empresa_sheets=None` em CADA iteração (XConnect e Aviation) → cada empresa processa TODAS as linhas
de despesa do Sheets. Efeitos: (a) despesa_op consolidada **2×** (537k vs 269k real); (b) **Aviation
NEGATIVO** (recebe todas as despesas com sua receita pequena); (c) XConnect subestimado.
Receita/CMV vêm de SQL filtrado por company (corretos) — só a despesa do Sheets (empresa_sheets)
estava com o bug. **FIX (1 linha): `empresa_sheets = company`** (sempre). Validado: `_filter_sheets`
filtra certo com "XConnect"/"Aviation" (soma = todas, sem órfãos). **Impacto:** XConnect 2026
153.900→**165.698**; Aviation −7.751→**+249.289**; consolidado 146.288→**414.986** (= soma real).
Reconcilia com a RENTABILIDADE (690k contribuição − ~269k overhead ≈ 415k DRE). **NÃO commitado**
(CFO quer validar). Aplicar e revalidar quando o CFO voltar. ⚠️ Standalone por empresa (empresa=
xconnect/aviation) JÁ estava certo (todas=False → filtra) — o bug é só na visão consolidada "todas".

## ⏳ NOVA RODADA DE REVISÃO DO DRE (CFO 13/jun, tarde) — reconciliar com a Rentabilidade
CFO acha os números do DRE 2026 estranhos:
- **DRE 2026: XConnect R$ 153.900,13 ; Aviation −R$ 7.751,14** (juntas ~R$ 146k).
- **Painel de RENTABILIDADE 2026 (ambas as empresas): lucro líquido R$ 690.089,07 (21,3% margem).**
  Esse painel JÁ deduz: comissão ML, CMV, comissão vendedor, frete, ICMS líquido, DIFAL, impostos
  federais (PIS/COFINS/IRPJ/CSLL) e ADS.
- **Gap ≈ R$ 544k** = a rentabilidade é margem de CONTRIBUIÇÃO (não inclui overhead); o DRE inclui
  as **despesas op/overhead** do Sheets. O CFO listou o que NÃO está na rentabilidade e deve estar
  no DRE: **pró-labore, salário do vendedor, fretes de envio para o ML, material de escritório,
  energia elétrica, impostos previdenciários, aluguel, despesas legais, ERP, internet, contador,
  reembolso.**
- **Tarefas da reconciliação:**
  1. **Reconciliar** Rentabilidade (R$ 690k) − overhead (Sheets despesas_op) = DRE (R$ 146k). Bater
     o número do overhead (~R$ 544k) e ver se faz sentido / não está inflado (dupla contagem).
  2. **Frete — NÃO é dobra (CFO esclareceu 13/jun):** o "frete" do overhead NÃO é o frete da venda
     ao consumidor (esse a rentabilidade já deduz). É o **frete de COMPRA (inbound dos insumos) +
     frete para o FULL (envio de estoque à logística do ML/fulfillment)** — fornecedor usual
     **Expresso Lajeado São Paulo**. São fretes DISTINTOS do frete de venda → **overhead legítimo do
     DRE, não dobra**. ⚠️ MAS: hoje "FRETE" está em `CATEGORIAS_EXCLUIR_DRE` (excluído de
     despesas_op por se assumir = frete da rentabilidade). **Conferir se o frete de compra/Full está
     numa categoria PRÓPRIA no Sheets (não "FRETE")** — se estiver em "FRETE", está sendo
     ERRADAMENTE excluído do DRE (buraco); se em categoria própria, ok. Idem comissão vendedor
     (rentabilidade já tira) vs algum lançamento no Sheets.
     **⚠️ NUANCE (CFO 13/jun): mesmo fornecedor (Expresso Lajeado SP) tem DOIS usos:** (a) **frete de
     entrega na HAVAN** → lançado EM CADA NF Havan → JÁ está na rentabilidade do canal (via
     `_havan_freight_per_unit`) → **NÃO é overhead, não somar de novo no DRE**; (b) **frete de
     compra (inbound) + frete para o FULL (ML)** → **overhead legítimo do DRE**. Ao mapear os
     lançamentos do Expresso Lajeado, SEPARAR os dois usos (pela obs/competência/produto): só o (b)
     entra como despesa op; o (a) é dedução da rentabilidade Havan (não duplicar).
     **✅ INVESTIGADO 13/jun (autônomo): a categoria FRETE é o INBOUND/FULL, e está ERRADAMENTE
     EXCLUÍDA.** FRETE XConnect 2026 = R$ 62.896, dominado por **EXPRESSO SAO PAULO LAJEADO R$ 52.879**
     + Expresso São Miguel R$ 7.576 + DHL/Braspress. = frete de COMPRA/FULL (o que o CFO descreveu),
     NÃO frete de venda. Hoje "FRETE" está em `CATEGORIAS_EXCLUIR_DRE` → some do DRE (buraco de
     ~R$ 78k: XC 62.896 + AV 15.320). **REFINADO (CFO + investigação 13/jun): há coluna `SUB-DESCRIÇÃO`
     que separa.** FRETE XConnect 2026: `HAVAN` R$ 18.400 (7 lanç — entrega Havan, JÁ na rentabilidade
     por NF → EXCLUIR); `FULL` R$ 7.200 + `(vazio)` R$ 37.296 (inbound/compra, ex.: "Frete 3000
     Ventosas ARX", "740 cases injetados" → MANTER). Aviation: R$ 15.320 inbound/Full, zero Havan.
     **Tratamento correto: NEM excluir FRETE inteiro (bug atual) NEM incluir inteiro — excluir SÓ as
     linhas com `SUB-DESCRIÇÃO`(norm)=='HAVAN'; manter FULL+inbound como despesa op.** Implementação:
     tirar FRETE de `CATEGORIAS_EXCLUIR_DRE` + regra que pula a linha FRETE quando SUB-DESCRIÇÃO=HAVAN
     (estilo `_is_anuncio_ml_sheets`). Impacto: +~R$ 44.496 (XC) + R$ 15.320 (AV) de despesa op (reduz
     resultado) — direção OPOSTA ao bug do "todas". ⚠️ Validar que as linhas `(vazio)` são todas
     inbound (não Havan sem tag) — as claramente Havan já estão taggeadas.
  3. **Mapear TODAS as categorias `DESCRIÇÃO DA CONTA` (coluna F do Sheets)** e classificar cada uma:
     já capturada na rentabilidade (excluir do DRE), overhead legítimo (manter), abaixo da linha, ou
     **ficou de fora** (não mapeada → buraco no DRE). Garantir que nenhuma categoria suma.
  4. Conferir se Aviation negativo (−7.751) faz sentido (pouca receita + overhead próprio?) ou se
     falta receita/sobra despesa.
  Ver [[ecommerce-rentabilidade-pedido]], [[ecommerce-overview-categorias]] (baldes de caixa).

**O que revisar:** se a DRE do ecommerce reflete competência de VENDA ou de ENTRADA para
CMV/insumos/créditos de imposto, e se isso precisa ser alinhado ao regime contábil (ou exibido nas
duas óticas, como fizemos com a cobertura 7d × histórica no painel Havan). Confirmar com o CFO
**qual aba/relatório DRE** exatamente (o ecommerce-agent tem rentabilidade/overview/cashflow; uma
"DRE" formal pode ser nova ou estar em planilha).

Relacionado: [[ecommerce-rentabilidade-pedido]], modelo de ICMS (ecommerce_icms_model.md),
[[ecommerce-overview-categorias]].

## ✅ FIXES APLICADOS 14/jun (manhã, COM o CFO — commits b225abe + anteriores)
Resultado DRE 2026 final após os fixes: **XConnect R$ 238.676 · Aviation R$ 224.082 · todas R$ 462.758.**
1. **"todas" dobrava despesa** (api.py:19845) — `empresa_sheets = company if not todas else None` → no
   consolidado cada empresa somava TODAS as despesas. Fix: `= company`. Aviation −7.751→+249k antes dos
   outros fixes. Commit junto do DIFAL (00b0168).
2. **DIFAL anual DOBRADO** (api.py:~19808) — no path anual `difal_val` mantinha o seed do SQL bruto
   (R$116k) e o loop somava +R$115k. Fix: `difal_val=0.0` antes do loop anual. DIFAL 231→115k (real
   R$116.053). resultado op subiu ~R$116k.
3. **FRETE** — só exclui SUB-DESCRIÇÃO=HAVAN (já na rentabilidade por NF); inbound/Full mantidos.
   Helper `_is_frete_havan`. FRETE volta: XC R$30.766, AV R$11.320.
4. **COMISSÕES — DECISÃO CFO: manter TODAS no DRE** (P&L de caixa real). O DRE só usa o CMV de
   compute_profitability, NÃO puxa a comissão da rentabilidade → não havia dupla, e excluir OMITIA
   custo real. (Investiguei a regra data-driven "excluir quem está no vendor_commission_config + Mark";
   mas a decisão foi manter todas.) COMISSÕES XC R$12.263, AV R$13.920. Removidas constantes órfãs
   `CATEGORIAS_EXCLUIR_DRE`/_NORM → `CATEGORIAS_EXCLUIR_DRE_SEMPRE_NORM` (só INSUMOS/ICMS).
   - Comissões mapeadas: XC = Rafael 5.262 (vendedor, config 15%) + Mark Barbouth 6.801 (Havan 1%) +
     Simone 200 (indicação). AV = Eduardo 12.220 ("50% lucro", NÃO no config) + Bernard 1.000 + Diego
     Estevam 700. `vendor_commission_config` tem Rafael/Diego Brunelli/Alessandro (NÃO Eduardo).
   - ⚠️ **PENDENTE config:** o `vendor_commission_config` aplica comissão da **Aviation ao Rafael 15%**
     (id=1), mas quem recebe comissão da Aviation no Sheets é Eduardo/Bernard/Diego — possível comissão
     "fantasma" do Rafael na rentabilidade da Aviation vs pagamento real ao Eduardo. **Validar.**
5. **Provisão clampava dias em 60** (api.py:17616) — UI oferece 90/120/180. Fix: `min(dias,180)`.
6. **DAS no DRE consolidado (frontend)** — `dre/index.tsx` mostrava DAS só no regime simples; no
   consolidado o DAS (R$67.607 Aviation) entrava no `tributos.total` mas não tinha LINHA. Fix: linha
   `['DAS (Simples Nacional)', d.tributos.das]` no array (aparece só quando >0). Rebuild do frontend.

## ⏳ NOTAS CFO 14/jun (A VERIFICAR — não implementado)
- ✅ **CMV ESTIMADO "32" — RESOLVIDO 14/jun (commit 139c146).** Era FALSO POSITIVO: as NFS-e
  (serviços) são adicionadas ao profit_df (frames, api.py:19710) para entrar na RECEITA, mas serviço
  não tem produtos → `cmv_source="sem_itens"` → contado como CMV estimado. XConnect não tem NFS-e
  (403)→0; **Aviation tem 32 NFS-e em 2026 → os 32**. O "todas" mostrava 32 porque vinha da Aviation
  (o subagente checou xconnect/aviation e errou a replicação da Aviation, reportou 0 — o endpoint VIVO
  dava aviation=32). Fix: `cmv_estimado_count` conta só mercadoria (exclui `_source=="nfse"`).
  cmv_total não muda (NFS-e=R$0 CMV). **PENDENTE menor:** todas todo-período ainda = 4 estimados REAIS
  de mercadoria (pré-2026) a investigar. (Obs: DRE call 19744 tem product_cost_history+tax_config_by_ym
  que /api/profitability 16061 NÃO tem — registrar p/ composições atípicas.)
- **ESTOQUE — onde trazer a informação de estoque** no painel (a definir onde/como).
- **ENDIVIDAMENTO (empréstimo) — onde trazer a informação de endividamento** (Sicredi etc.) no painel.
  Ver [[ecommerce-sicredi-emprestimo]], [[ecommerce-cashflow-financiado]]. (O principal de empréstimo
  agora sai do DRE via `movimentos_financiamento` — destino natural é uma visão de Endividamento + Fluxo.)

## ✅ FIXES 14/jun (tarde, COM o CFO — auditoria rentab×Sheets×DRE)
Auditoria (subagente, ponte numérica fechou R$0,01/0,02 validada no endpoint vivo). Resultado 2026
FINAL: **XConnect líquido R$ 176.254 · Aviation R$ 237.040 · consolidado R$ 413.294.**
1. ✅ **DUPLA contagem comissão vendedor NF-e (commit 7b19ced).** `bling_nfe.comissao` é comissão de
   VENDEDOR (não plataforma, vide metrics.py L384) mas entrava nas DEDUÇÕES (com_nfe) E na despesa_op
   via Sheets COMISSÕES. Caso: Eduardo NF Aviation 000041 R$12.220,55 = Sheets "COMISSÃO 50% LUCRO".
   Fix: deduções = só com_ml + fretes (removido com_nfe). Aviation 224k→237k (sai tb Rafael NF R$803
   sem par no Sheets — se pago, lançar). XConnect inalterado (com_nfe=0).
2. ✅ **RESULTADO FINANCEIRO separado (commit 7365dc6) — decisão CFO: opção "separar".** Antes a linha
   somava a categoria `financiamento` INTEIRA (principal empréstimo +917k XC, RF -300k, transferências
   + juros) e era só informativa. Agora: **Resultado Financeiro = rendimento RF − (JUROS + IOF +
   tarifas bancárias)**; IOF/tarifas saíram do opex p/ despesa financeira; **compõe o líquido**
   (`resultado_liquido = resultado_op + resultado_financeiro`). Principal/transferências →
   `abaixo_da_linha.movimentos_financiamento` (fluxo de caixa, fora do resultado). Impacto: XConnect
   líquido 238k→176k (juros -62k); IOF/taxas trocam bucket (net zero). Frontend: nova seção "Resultado
   Financeiro"; tipos DreData (resultado_financeiro{receitas,despesas,total}; abaixo_da_linha trocou
   resultado_financeiro→movimentos_financiamento). Constantes `_JUROS_NORM`, `_DESPESA_FINANCEIRA_NORM`.
3. ✅ **DAS no consolidado** — em AMBOS os cards (TributosDetail + painel lateral). Commits 76f30ea+3ef36b7.

## ✅ FIXES 14/jun (noite — apresentação dos tributos/DIFAL)
- ✅ **DAS no card tributos do consolidado** (commits 76f30ea/3ef36b7).
- ✅ **DIFAL é TRIBUTO, não dedução (commit a4deef6) — decisão CFO: "ele é parte do ICMS".**
  Inconsistência: ICMS líq estava em Tributos mas DIFAL em Deduções. Agora DIFAL entra em
  `tributos.total` (junto do ICMS líq), sai das deduções (`deducoes_receita.difal=0`), gráfico mensal
  não soma DIFAL no ded (flui via `_trib_rate`). **Resultado líquido INALTERADO** (só muda de lugar);
  EBITDA sobe (DIFAL fica depois dele — EBITDA é antes de impostos); tributos.total consolidado
  374.841→490.154 e a **soma das linhas de tributos volta a FECHAR com o total** (era a queixa do CFO:
  somava 490.154 e o card mostrava 374.841 — a diferença era o DIFAL listado mas fora do total).
- Resultado 2026 (inalterado pelos fixes de apresentação): **XConnect líq R$176.254 · Aviation
  R$237.040 · consolidado R$413.294**; EBITDA consolidado R$980.190; tributos R$490.154.

## ✅ IMPOSTO HÍBRIDO — DECISÃO FINAL (commit facd049, 14/jun noite) — SUPERA o "Fix A"
A dupla de imposto (computado em tributos + Sheets em despesa_op) foi resolvida assim: **o DRE usa o
imposto LANÇADO no Sheets** (guia do contador = valor REAL, por competência, inclui parcelamento) —
**mais apurado** que o computado (subestimou ~R$184k em 2025: real 662k vs computado 478k no XConnect).
O Sheets também alimenta o fluxo de caixa. **Híbrido POR EMPRESA:** LANÇADO onde houver guia; meses
decorridos sem guia (defasagem ~1 mês) → PROVISIONA pelo computado. **Aviation NÃO lança Simples**
(R$1.618 em 2025, 0 em 2026) → cai 100% no computado/provisão. Impl: `CATEGORIAS_TRIBUTO_NORM`
(SIMPLES NACIONAL/DAS/PIS/COFINS/IRPJ/CSLL) → `imposto_lancado` por competência (não despesa op);
`provisao = computado_federal × (meses_sem_guia / meses_decorridos)`; `tributos.total = lançado +
provisão + ICMS/DIFAL(LP)`; campos `imposto_lancado/provisao_imposto/imposto_computado_federal`. Chart
exclui tributo do desp (flui via `_trib_rate`). Frontend: "Lançado + Provisão + ICMS/DIFAL = Total"
(soma FECHA, validado vivo 2025/2026). 2025 todas líquido →720k, 2026 →399k. INSS/IMPOSTOS(importação)
ficam como custo real (não entram). **Pendências:** (1) LP com lançamento PARCIAL (XConnect 2026 só
PIS/COFINS, sem IRPJ/CSLL) — provisão aproxima, revisar precisão; (2) ICMS LP segue computado.

## 🔴 ITEM C — BACKFILL ML HISTÓRICO: IMPOSSÍVEL (conclusão CFO 14/jun)
- **A API do ML (`/orders/search`) tem janela móvel de ~365 dias** (não só o ADS, como se pensava).
  Testado: hoje 14/jun/2026 → retorna pedidos só a partir de ~14/jun/2025; mai/2025 e tudo antes = 0,
  **por QUALQUER tamanho de janela** (mês/semana/dia) e tb no `/orders/search/archived`. Confirmado com
  controle: jun/2025 mês todo=608, mas jun/2025 semana1 (antes do corte)=0.
- **`ml_orders_sync` XConnect: 11.546 pedidos, todos 2025/2026; mais antigo 24/03/2025** (ID
  2000011120571580). Zero em 2024. IDs ML são sequenciais → não há pedido antigo deletado. Varredura em
  TODAS as tabelas: nov/dez 2024 só existe em `bling_nfe` (235 NF-e **B2B** — Construtora/Marfrig/JET
  LINE, CFOP 6102/6108, ticket mediano R$1.279). Essa receita B2B JÁ está no DRE (não falta).
- **CONCLUSÃO CFO (opção 2):** houve venda ML em nov/2024–fev/2025 que **nunca foi sincronizada** (1ª
  sync rodou só em mar/2025) → **dados ML iniciais PERDIDOS na origem, irrecuperáveis** (API não volta
  tanto). Não há backfill possível. **Ação:** marcar 2024 + jan-fev/2025 como "ML incompleto/dados
  parciais" no DRE (período anterior ao 1º pedido ML sincronizado = 24/03/2025). 2024 = só B2B (NF-e),
  é jovem (2 meses) — margem alta de B2B é real, não artefato. Alternativa não explorada: relatórios de
  repasse/settlement do ML podem ter o financeiro histórico (outra integração).

## ✅ ITEM C — RESOLVIDO (15/jun, commit 95f7fed) — ML estimado via repasse
Backfill via Bling DESCARTADO: loja **X Connect Full NÃO tem NF-e no Bling** (CFO confirmou direto no
Bling; minha coleta da loja Full deu 0). E o ML drop-off (loja 205212252) só viria parcial. Dados ML
do início são **irrecuperáveis** na origem. **Solução final (decisão CFO):** para meses SEM
ml_orders_sync (XConnect dez/24–fev/25), estima a receita ML pelo **repasse LÍQUIDO do Sheets**
("VENDA DE PRODUTOS" favorecido **X CONNECT IMPORT**) brutado pelas razões do mês-base **abril/2025**:
taxa ML **17,0%** + frete **5,1%** → bruto = repasse/**0,779**; **CMV 31,7%** do bruto. Helper
`_estimate_ml_from_repasse` + constantes `_ML_EST_*` (api.py); soma em rec_ml/com_ml/frete_ml/cmv; só
XConnect; meses com ml_orders_sync usam o real (sem dobra). Campo `ml_estimado` na resposta + aviso
âmbar no frontend. **ESTIMATIVA ao vivo (NÃO grava no banco).**
**Resultado consolidado FINAL:** 2024 receita R$927k / líquido R$398k (43%, jovem/B2B-heavy + ML est.);
2025 receita R$4,94M / líquido R$925k (18,7%); 2026 R$3,25M / líquido R$380k (11,7%).
**Pendências:** corrigir bug de produção do param de data do Bling (`dataEmissaoInicio`→`Inicial`,
agent/bling.py); 2024 margem ainda alta (período jovem). Repasse abril (mês-base) tem ML+Full juntos.

## (descartado) ITEM C — tentativa via Bling — janela API 365d + Full sem NF-e
**🐛 BUG DE PRODUÇÃO encontrado:** `BlingClient.list_nfe`/`list_nfe_all` (agent/bling.py ~289-292) usa
`dataEmissaoInicio`/`dataEmissaoFim` — **a API v3 IGNORA** (param errado) e devolve as NF-e MAIS
RECENTES. O correto é **`dataEmissaoInicial`/`dataEmissaoFinal`** (+ aceita **`idLoja`**). Sintoma: pedir
dez/2024 devolvia jun/2026. Afeta qualquer busca histórica (o sync diário não percebe pois quer as
recentes). **CORRIGIR no código** (e o monitor de NF-e pendentes scheduler ~3122 depende disso).
**Teste LIMPO confirmou:** `dataEmissaoInicial=2024-12-01&dataEmissaoFinal=2024-12-31&idLoja=205258337`
→ 100 NF-e TODAS de dez/2024 ✓. **As NF-e ML de dez/24 SÃO recuperáveis.** (Sem idLoja → 504 timeout: a
query ampla é pesada; sempre filtrar por loja.) ⚠️ TODOS os números anteriores de "dez/2024" (700/1984
NFs etc.) estavam ERRADOS — eram 2026 recentes, pelo param bugado.
**Lojas ML:** 205212252 + 205258337 (= X Connect ML + X Connect Full); B2B = loja 0/série 1.
**BLOQUEIO ATUAL:** API Bling muito lenta/instável (44% das requests = 504); buscar milhares de
`get_nfe_details` (valorNota não vem na lista) é inviável agora. **PLANO:** (1) corrigir o param no
bling.py; (2) backfill robusto (param correto + idLoja + retry no 504) buscando dez/24→23/03/25 das 2
lojas ML, autorizadas (situacao 5/6), parse `det.get('data',det)`; (3) VALIDAR bruto×repasse por mês
(dez R$362.598 líquido; taxa esperada ~13-17%+frete, NÃO 33%) e mostrar ao CFO ANTES de gravar; (4)
persistir em bling_nfe (situacao=6, +campo loja) só dos meses sem ml_orders_sync (sem dobra). Rodar
quando a API estiver saudável.

## (histórico) ITEM C — análise inicial (parcialmente baseada no param bugado, ver acima)
A receita ML do período perdido (dez/24–fev/25) **É recuperável pela API do Bling** (sem limite de 12
meses, diferente do ML). O repasse no Sheets confirma a venda: VENDA DE PRODUTOS favorecido
**"X CONNECT IMPORT"** — dez/24 R$362.598, jan/25 R$164.000, fev/25 R$180.500 (líquido).
- **Bling tem 1.100 NF-e em dez/2024 (nosso DB só 161).** Split por LOJA: `205212252` (277 NFs) +
  `205258337` (815 NFs) = canal ML (**X Connect ML + X Connect Full**, série 2, tickets pequenos
  R$338-1041); `loja 0` (8, série 1, tickets grandes) = B2B (o que já temos). **Nosso sync do Bling
  filtra fora as lojas ML** (provavelmente p/ não dobrar com ml_orders_sync no período recente).
- Bruto ML dez/24 ~R$546k → líquido ~R$366k ≈ repasse R$362k ✓ (casou). `bling_nfe` NÃO tem campo loja.
- **PLANO:** backfill via `BlingClient.list_nfe_all(data_inicio,data_fim,pagina)` (paginar) das lojas
  205212252/205258337, dez/24–fev/25, só AUTORIZADAS (conferir situacao — lista traz situacao=5; nosso
  DB usa 6; 1.100 inclui canceladas/rascunho, filtrar); `valorNota` via `get_nfe_details` (parse:
  `det.get('data',det)`). Usar como receita ML BRUTA desses meses (sem ml_orders_sync→sem dobra; ML só
  começa 24/03/2025). Adicionar campo loja no bling_nfe ou tabela própria. SUPERA o gross-up.
- ⚠️ Volume grande (~1.100 NF/mês × 3 meses, detalhes 1-a-1 = lento). One-time backfill. **NÃO gravar
  sem casar o total bruto com o repasse por mês primeiro.**

## ⏳ A INVESTIGAR (achados 14/jun, NÃO resolvidos)
- 🔴 **REGIME competência×caixa nas DESPESAS — anual exclui despesas A VENCER (resolve o gap de 227k).**
  ANO=1899 **NÃO é bug do Excel: é status "A VENCER"** (CFO 14/jun) — a coluna ANO vem da data de
  PAGAMENTO (vazia enquanto não pago → 1899). O loop ANUAL (api.py ~19878) `if 0<ANO<2000: continue`
  exclui as despesas A VENCER → **despesa por CAIXA**, enquanto a **RECEITA é por COMPETÊNCIA** (data
  NF/venda) → INCONSISTENTE. O gráfico mensal NÃO filtra → conta R$262.869 a mais (XConnect 2026),
  incluindo despesas reais a vencer: COMERCIAL R$49.000 (salário Rafael jun, a vencer), DESPESAS LEGAIS
  R$5.417, DESPESAS COM VENDAS R$5.000 + tributos lançados (PIS R$4.774 + COFINS R$22.035 + ICMS
  R$107.841 + INSUMOS R$68.802 — esses 2 já saem por SEMPRE). **Decisão CFO pendente: DRE por
  COMPETÊNCIA** (incluir a vencer pela competência — consistente com a receita) **ou CAIXA** (só pagas
  — mas então a receita tb teria que ser caixa). **Fix técnico junto:** excluir do despesa_op as
  CATEGORIAS DE TRIBUTO lançadas no Sheets (PIS/COFINS/IRPJ/CSLL/DAS/DIFAL — já computados em tributos;
  hoje só ICMS sai sempre) p/ não dobrar. Unificar loop do gráfico mensal com o anual. Afeta TODOS os anos.
### 🔴 AUDITORIA DRE 2024 e 2025 CONSOLIDADO — RESULTADO (14/jun, 2 subagentes validados no endpoint vivo)
**Veredito: nenhum dos dois é confiável — dados incompletos + bug sistêmico de dupla contagem de imposto.**

**2024 — INUTILIZÁVEL (só Nov-Dez existem):** ML sync começa **24/mar/2025** → ZERO pedidos ML em 2024;
ADS começa dez/2025; ml_nfe só 2026. Receita 2024 = só NF-e Bling Nov+Dez (R$461.922), deduções=0,
ADS=0 → margem bruta "74,7%" fisicamente impossível (real ~46%). 10/12 meses zerados. NÃO usar p/ decisão.

**2025 — 2 meses de ML faltando + dupla de imposto:** jan/fev sem receita ML (sync só desde mar/25; têm
NF-e mas não ML) → jan/fev com EBITDA negativo artificial. ADS API só cobre Dez (R$3.108); resto vem do
Sheets. Líquido reportado R$386k (8,6%) subestimado.

**🔴🔴 BUG SISTÊMICO CRÍTICO — DUPLA CONTAGEM DE IMPOSTO (afeta todos os anos).** O DRE computa o imposto
em `tributos` (DAS no Simples; PIS/COFINS/IRPJ/CSLL/ICMS no LP) E as contas de tributo lançadas no Sheets
("SIMPLES NACIONAL", "IMPOSTOS", "INSS"/"PREVIDENCIÁRIO", "PIS", "COFINS", "DAS"...) caem em despesa_op,
reduzindo o EBITDA → imposto contado **2×**. `CATEGORIAS_EXCLUIR_DRE_SEMPRE_NORM` (api.py:214) só exclui
INSUMOS e ICMS. Impacto: **2025 líquido subestimado ~R$410-546k** (DAS computado 478k + Sheets SIMPLES
~537k); 2024 R$38.780. **Em 2026 está LATENTE/mascarado:** os PIS/COFINS do Sheets XConnect 2026 estão
A VENCER (ANO=1899) → o filtro ANO<2000 os exclui por acaso; quando forem pagos, começam a dobrar.
**Fix:** adicionar as categorias de TRIBUTO à exclusão sempre (decidir fonte única: imposto PAGO do Sheets
OU COMPUTADO pelo regime, nunca ambos). + DAS anual soma alíquota mês a mês (hoje aplica a de 1 mês sobre receita do ano).

**🟡 a-vencer (ANO=1899) — análise CFO "competência futura vs atual" (XConnect 2026):** é MISTO.
- Competência FUTURA (Rafael salário Jul-Dez R$42.000): projeção de despesa futura → exclusão DEFENSÁVEL
  (a receita também não é projetada nesses meses). NÃO é bug.
- Competência ATUAL/≤ mês corrente (R$44.226): PIS 4.774 + COFINS 22.035 (tributos → saem de qualquer
  forma, ver dupla acima) + COMERCIAL 7.000 + DESPESAS LEGAIS 5.417 + DESPESAS COM VENDAS 5.000. Destes,
  **~R$17.417 são despesa legítima de competência atual sendo WRONGLY excluída** (deveria entrar por competência).
- 2025: ANO<2000 dropa R$212.445 (SIMPLES 126.917 + INSUMOS 72.628 + **FRETE R$12.900 legítimo perdido**).
**Conclusão:** o filtro ANO<2000 é um proxy cru — acerta ao não projetar despesa futura e ao mascarar
tributos a vencer, mas erra ao dropar despesa legítima de competência atual a vencer. Fix correto:
separar (a) excluir tributos sempre; (b) incluir despesa legítima por COMPETÊNCIA (não por pagamento);
(c) no mês corrente/futuro, alinhar com a ótica da receita (se receita é realizada, não projetar despesa futura).

**🟡 ADS cobertura parcial:** a regra exclui anúncio do Sheets nos meses cobertos pela API e usa o valor
da API — mas em meses de cobertura PARCIAL (ex.: Dez/2025 API R$3.108 vs Sheets R$20.312) subconta ~R$17k.
- 🔴 **Gráfico mensal do DRE NÃO reconcilia: gap ~227k no EBITDA** (XConnect: EBITDA anual 560.225 vs
  soma mensal 333.189; RECEITA bate R$0,00). PRÉ-EXISTENTE (não é da mudança de res.financeiro). A soma
  dos meses do `chart_mensal` ≠ anual → resultado_liquido mensal não confiável. Causa provável:
  deduções/CMV/tributos mensais (rate-based / SQL separado) divergem do cálculo anual. Investigar.
- 🟡 **"IMPOSTOS" (XConnect R$14.806) = imposto de IMPORTAÇÃO** (despachante PORTIQ, FEDEX aéreo imãs,
  envio Colômbia), NÃO tributo federal. Hoje em despesa_op (overhead). Contabilmente deveria ser custo
  de nacionalização (CMV/landed cost) junto com INSUMOS. Não é dupla, mas desloca margem bruta↔op.
- 🟡 **Comissão Aviation "fantasma":** vendor_commission_config aplica 15% ao Rafael na Aviation, mas
  quem recebe comissão Aviation no Sheets é Eduardo/Bernard/Diego. Validar.
- 🟡 **REVISAR a REGRA de registro das comissões pagas aos vendedores (CFO 14/jun).** Hoje há TRÊS
  fontes da mesma comissão e a regra ficou espalhada: (1) `vendor_commission_config` (auto 15% s/ lucro
  — aplicada SÓ no painel de Rentabilidade — Rafael/Diego Brunelli/Alessandro); (2) **Sheets COMISSÕES**
  (pagamento real — MANTIDO INTEIRO na despesa_op do DRE, decisão de hoje); (3) `bling_nfe.comissao`
  (comissão lançada na NF — REMOVIDA das deduções hoje p/ não dobrar). Definir a fonte ÚNICA de verdade
  por contexto (DRE usa Sheets; Rentabilidade usa config auto), padronizar onde cada comissão é lançada
  (Eduardo/Bernard/Diego não estão no config; Rafael da NF não está no Sheets) e documentar a regra
  para não reabrir dupla/buraco. Casa com a "comissão fantasma" da Aviation acima.
- 🟢 **4 CMV estimados reais** (mercadoria pré-2026, fora de 2026). Investigar quais.
- 🟢 **cost_ref_date divergente** profitability(01/01) vs DRE(31/12) → CMV difere ~62k entre os dois
  endpoints (DRE internamente consistente). Alinhar se quiser que batam exatamente.

## ✅ VEREDITO CUSTO EM DOBRO (15/jun) — a suspeita NÃO se confirma no resultado operacional
Auditoria read-only de `get_dre` (api.py ~19855-20400). **NÃO há dupla contagem de CMV/INSUMOS.**
- CMV entra UMA vez, só via `compute_profitability` (api.py ~20136-20150) — regime de competência, custo por pedido. Não há fallback de CMV via Sheets dentro do DRE.
- INSUMOS (saída de caixa da COMPRA) é EXCLUÍDA do opex: `CATEGORIAS_EXCLUIR_DRE_SEMPRE_NORM = {INSUMOS, ICMS}` (api.py:216), aplicada no laço de despesas (`continue` em ~20290). É o mecanismo anti-dobra.
- Números confirmam: INSUMOS pago/ano (422k/2,3M/2,0M) NÃO aparece no opex do DRE em nenhum ano; CMV no DRE é linha separada (264k/1,85M/1,37M). INSUMOS (caixa, compra) ≠ CMV (competência, venda) — magnitudes diferentes por timing de estoque, o que é correto.
- A reposição de corpos/imãs como saída de caixa é do **Fluxo de Caixa** (regime caixa), módulo separado — não toca o resultado operacional do DRE.
- **A dobra que EXISTIA e já foi corrigida** era nas DESPESAS_OP do Sheets (DIFAL/IMPOSTOS/MARKETING dobrando deduções/tributos/ADS) — tratada nos commits de "fix dupla contagem". O CMV/insumo (suspeita central do CFO) está limpo.

## ✅ B2B 2026 CMV ~55% (investigado 15/jun) — REAL, não bug. Causa: HAVAN entrou no B2B
Reproduzido via `compute_profitability` (`_source='bling_manual'`), todos `cmv_source='real'` (sem SKU vazio/NaN/cross-join/comissão no CMV):
- 2025: 264 pedidos, receita R$ 1,23M, CMV 24,6% (mix premium ML, ticket pequeno, margem alta).
- 2026: 59 pedidos, receita R$ 1,21M, CMV 57,5%. **HAVAN = 9 NFs, R$ 966k (80% da receita B2B), CMV 61,8%.** B2B avulso (50 ped., R$242k) segue saudável a ~40% CMV.
- Havan ~62% é compressão REAL de margem (atacado: cabos/suportes plásticos a preço baixo — Case c/ imãs 295/114, cabo USB-C 50/32, etc.). Custos versionados corretos (China 2026).
- Desconto contratual Havan (NF 561 override) NÃO infla o CMV% — denominador do summary é receita BRUTA, não líquida.
- **Recomendação (não implementada):** segmentar o canal B2B em "Havan" vs "B2B avulso" no dashboard, senão a Havan arrasta a margem agregada e esconde que o avulso continua ~40%.
