---
name: ecommerce-cashflow-mes-vigente-realizado
description: Pendência (ecommerce-agent) — mês vigente do fluxo de caixa/gráfico/provisão não desconta dividendos e despesas op JÁ PAGAS no mês
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

**PENDENTE (anotado 12/jun/2026, a pedido do CFO — PARA ESTUDARMOS, nada codado):** ao analisar o
**fluxo de caixa financiado**, o CFO simulou e notou que **dividendos já pagos no mês** e
**despesas operacionais já pagas no mês** parecem **não ser descontados da projeção do mês
vigente**. Melhoria a estudar abrange **fluxo de caixa + gráfico + provisão**.

**REVISÃO (12/jun) — a suspeita procede.** Em `get_cashflow` (`agent/api.py`, loop ~api.py:4984+,
montagem do `saldo_periodo_base` ~api.py:5121, dividendos ~api.py:5141):
- Partida: `saldo_base_proj = saldo_real + vencido_receber − vencido_pagar`. `saldo_real` = soma
  all-time das linhas efetivadas = banco real hoje → **já reflete** dividendos e opex já pagos no
  mês (saíram do banco).
- **(1) Dividendos já pagos — DEFEITO claro.** O mês corrente projeta uma distribuição NOVA:
  `dividendos = min(excesso × 0,80; R$120k)` (`_DIV_PERC=0.80`, `_DIV_MAX=120000`,
  `_DIV_BUFFER_OP=80000`), SEM checar dividendo já distribuído no mês. O já-pago reduz o excesso
  (autocorreção só PARCIAL), mas o modelo ainda pode projetar até +R$120k por cima de uma
  distribuição que já ocorreu. Falta reconciliar com o dividendo real do mês (ex.: no mês corrente,
  usar o já-pago e só modelar o residual até o teto; ou suprimir a modelagem no mês vigente).
- **(2) Opex já pago — risco de DUPLA SUBTRAÇÃO nas estimativas modeladas.** `a_pagar_op` do mês
  corrente só pega PENDENTES (STATUS A VENCER/A PAGAR; VENCIDO vai p/ vencido_pagar) — opex já pago
  (efetivado) corretamente fora dele e já no saldo_real. PORÉM o mês corrente também subtrai
  estimativas modeladas independentes de status de pagamento: `impostos_est`, `ads_est`,
  `cmv_base`, `cmv_imas`, `cmv_varejo88`. Se parte JÁ foi paga no mês (já no saldo_real), subtrai
  2×. Comentários no código alegam tratar ADS-vencido (≤hoje fora) e impostos do mês corrente
  (=base mês anterior fechado); o suspeito principal é o **CMV (cronograma por data de embarque
  +30/60d)**, que não olha pagamento real.

**Causa-raiz conceitual:** o mês vigente MISTURA realizado (`saldo_real`) com estimativas de
**mês cheio** sem subtrair o que já foi realizado DENTRO do próprio mês. Padrão correto p/ mês
parcial: para cada rubrica, projetar só o RESIDUAL = estimativa_mês − já_realizado_no_mês (piso 0),
ou partir de saldo de início-de-mês e somar realizado+residual.

**A estudar também:** replicar a verificação na **Provisão** (`/api/provisao`) e no **gráfico** —
ver se o mesmo viés de mês-parcial existe lá. Ver [[ecommerce-overview-categorias]] (baldes de
caixa) e [[ecommerce-provisao-pontos-cegos]].

## ⏳ BUG (CFO 13/jun, A VALIDAR) — Provisão: filtro de 120 dias travado em 60 no backend
A UI da Provisão (`ProvisaoCard.tsx`) oferece **120 dias** (`<option value={120}>`), mas o backend
`get_provisao` (api.py:17616) faz `dias = max(7, min(dias, 60))` → **clampa em 60**. Selecionar 90 ou
120 não muda nada além de 60. Com a integração Havan a faturar (recebimentos Out-Dez, impostos/CMV
Jun-Set), a 120d apareceria MUITO mais no day-by-day. **Fix:** `min(dias, 120)` (e conferir que as
queries/projeção aguentam 120d sem degradar). Validar com o CFO. Afeta tb a opção 90 dias.

## 📋 AUDITORIA AUTÔNOMA cashflow+provisão (14/jun madrugada) — achados a validar
1. **[ALTA] Dividendos do mês vigente em DOBRO** — `saldo_real` já reflete R$ 71.307 de abaixo_linha
   (dividendos/capex/pró-labore) JÁ pagos em junho, MAS o loop distribui dividendo novo
   `min(excesso*0,8;120k)` (L5303) sem abater o já pago. Mascarado (junho negativo→0); dispara em mês
   de superávit. Fix: abater abaixo_linha já pago do orçamento OU pular distribuição no mês corrente.
2. **[ALTA] Provisão clampa `dias` em 60** (L17616 `min(dias,60)`) mas a UI oferece 90/120/180 → todas
   retornam 60 sem aviso. Fix: `min(dias,180)` (validar perf dos loops por dias) ou limitar o dropdown.
3. **[MÉDIA] Burn rate / net_burn / projeção contaminados pelo mês PARCIAL** — `ultimos_3=historico[-3:]`
   (L3586) inclui o mês corrente parcial → subestima receita ~27% e despesa ~17%. Afeta `burn_rate`,
   `net_burn` e o array `projecao` (3 meses). `runway_months` está OK (usa último mês completo). Fix:
   usar `historico_completo[-3:]`.
4. **[MÉDIA] Provisão SEM a guarda anti-dupla de ADS que o cashflow tem.** `get_contas_pagar_por_dia`
   inclui qualquer A VENCER sem filtrar anúncio ML, e em paralelo soma `_ads_invoice_schedule`. Hoje 0
   ADS lançado como A VENCER (sem dobra ativa), mas latente. Fix: replicar `_is_anuncio_ml_sheets`→skip.
5. **[MÉDIA-BAIXA] Base de imposto da PROVISÃO ignora `exclude_from_revenue`** (L18211/18217/18231/18237)
   — diferente do cashflow. R$ 11.412 em 8 NF-e excluídas (ex.: 81-87 favor pessoal) entram na base de
   PIS/COFINS/IRPJ da provisão. Fix: adicionar o filtro (paridade com cashflow).
6. **[MÉDIA-BAIXA] `receita_havan_est` (recompra futura) entra BRUTA, sem imposto/CMV** (só os pedidos
   ABERTOS têm imposto/CMV casados). Pequeno hoje (R$ 15k em 2027). Fix: modelar imposto+CMV ou documentar
   como teto otimista.
7. **[BAIXA] `historico` por `timedelta(days=30*i)`** (L3509) — frágil, drift ~5d/ano pode pular/duplicar
   mês. Fix: iterar por mês de calendário.
   ⚠️ **Caveat ML repasse mês corrente:** o filtro de "ML já recebido" casa R$ 46.650, mas há R$ 60.306
   efetivado no mês — os R$ 13.656 restantes (B2B/Shopify/repasse sob outro favorecido) podem se sobrepor
   ao estimado por cima (dupla contagem parcial ~R$ 13k). CFO confirmar se é outro canal (sem overlap).
   ✅ Verificados OK: override fiscal×a_pagar_op, CMV Havan disjunto (cmv_base só ML), IRPJ/CSLL provisão,
   retenção empréstimo ML (cap 25%), runway, tax/ads_daily display-only.

## ⏳ PENDENTE (CFO 13/jun) — revisar o indicador de BURN RATE (mensal)
Revisar o cálculo do **burn rate mensal** no fluxo de caixa. Hoje no `get_cashflow` (api.py):
`burn_rate_bruto = media_desp` (média de despesas dos últimos 3 meses do histórico) e
`net_burn = media_desp − media_rec`; `runway` usa o último mês COMPLETO (exclui mês parcial). Suspeita
do CFO de que o indicador pode não fazer sentido — provavelmente ligado aos mesmos vieses do mês
vigente (mistura realizado/estimativa) e à base do histórico (média 3m vs ritmo real). Conferir a
fórmula, a base (3m vs mês fechado) e se reflete o burn real (despesas − receitas recorrentes).

## ✅ INVESTIGAÇÃO FECHADA (15/jun) — mês corrente: dupla subtração só em IMPOSTOS e DIVIDENDOS
Auditoria read-only de `get_cashflow` (modelo é **aditivo sobre saldo_real**, não "mês cheio por cima"):
- **CMV — LIMPO (refuta suspeita anterior).** `_base_payment_schedule`: 1º pagamento = `now + next_order_days + 30/60d` → cai em **julho**, nunca no mês corrente (api.py ~4705-4718). CMV imãs em datas fixas (ago/mar-2027) e gated `enabled=False`. NÃO subtrai 2× no mês vigente.
- **ADS — LIMPO.** `_ads_invoice_schedule` só pega faturas `vencimento ≥ hoje`; gasto real até ontem + média×dias restantes. Vencido já está no saldo de partida. Prorrateado correto.
- **IMPOSTOS — dupla subtração CONFIRMADA (não estrutural).** Estimativa CHEIA do mês corrente (base = mês anterior fechado) só é suprimida se o contador lançar o real como linha **PENDENTE** (`_tax_overrides`). Imposto já **PAGO** no mês (saiu de pendentes, já rebaixou saldo_real) NÃO tem override → estimativa cheia continua subtraindo → 2×. Exceção correta: DIFAL usa residual. Impacto: pessimista em dezenas de milhares/mês (ex.: ICMS RS ~R$27k).
- **DIVIDENDOS — não reconciliado (confirma item 1 acima).** Mês corrente sempre re-modela `min(80%×excesso;120k)` sem ler o já-distribuído no mês → saída modelada adicional à já-paga.
- **Direção:** projeção do saldo do mês corrente fica **PESSIMISTA**.
- **Fix recomendado (não implementado):** aplicar a lógica residual do DIFAL aos demais impostos (subtrair já-pago efetivado por tipo, projetar `max(0, cheia − já_pago)`) + reconciliar dividendo (`max(0, modelado − já_pago)`, helper de dividendo efetivado já existe no overview ~api.py:1906-1909). NÃO tocar CMV/ADS.

Relacionado: [[ecommerce-dre-competencia-revisar]] (auditoria DRE em curso),
[[ecommerce-cashflow-financiado]], [[ecommerce-sicredi-emprestimo]].

## ✅ IMPLEMENTADO (16/jun) — guarda anti-dupla INSUMOS lançados vs CMV modelado
INSUMOS lançado (a vencer) já entra no `a_pagar_op`; o CMV modelado (`_havan_aberto_cmv` Havan +
`_base_payment_schedule` ML) projetava os mesmos insumos → dobra. Fix (commit `8b0c455`):
- Helpers `_insumo_lancado_avencer_por_sub(rows, empresa)` (soma INSUMOS a-vencer por **SUB-DESCRIÇÃO**
  do Sheets — o CFO separa cabo/ventosa/cases lá) + `_abate_insumo_modelado` (consome do mês mais cedo,
  piso 0, dentro da janela) + `_abate_schedule_total` (ML).
- **Havan** (`_havan_aberto_cmv_lancamentos`): abate **VENTOSAS + CABOS**; subtipo **corpo** (cases Havan
  120d) fica INTACTO — o "CASES" do Sheets é ML, não Havan.
- **ML** (`_base_payment_schedule`, 30/60d): abate **CASES**.
- Aplicado no **fluxo (get_cashflow) E na provisão (get_provisao)**.
- Corpos Havan: faturamento ≈ entrega−5d → pagamento **entrega+115d** (era +120; ajuste fino).
- Validado: fluxo `havan_aberto_cmv` 303.548 → **210.345** (−93.203 dobra: ventosa 31.720 cheio + cabo
  61.483 parcial). Custo de cabo no modelo = **R$25/un flat** (custo-alvo Havan, ainda sem crédito ICMS).

## ⏳ AINDA PENDENTE — DIVIDENDOS do mês vigente em DOBRO (mesma família, NÃO corrigido)
A guarda de insumos NÃO cobre dividendos. No mês corrente o modelo re-distribui dividendo fresco
`min(excesso×0,80; 120k)` (api.py:5479-5485) SEM abater o que já foi pago no mês — e o `saldo_real`
de partida JÁ reflete o dividendo pago. → **dupla subtração no mês vigente** (mesmo no fluxo e provisão).
Fix proposto: no mês corrente, `dividendo = max(0, modelado − já_pago_no_mês)` (ler DIVIDENDOS efetivado
do Sheets; helper de dividendo efetivado já existe no overview ~api.py:1906-1909). A pedido do CFO 16/jun.

## ✅ IMPLEMENTADO (16/jun) — dividendo do mês vigente não conta em dobro (commit d25d463)
Mês corrente: `dividendos = max(0, modelado − já_pago)` onde já_pago = DIVIDENDOS **efetivado** no mês
(helper `_abaixo_linha_pago_mes(rows, empresa, "DIVIDENDOS", ano, mes)`, casado pela DATA de pagamento).
Total distribuído no mês = max(modelado, já-pago) → respeita o teto de 120k e nunca subtrai 2×. Meses
futuros inalterados. Só no **cashflow** (a provisão não modela dividendos).
**Sweep pró-labore/CAPEX:** NÃO são modelados no forward (pró-labore = despesa real coberta pelo
`_DIV_BUFFER_OP`=80k; CAPEX só vem do Sheets) → **sem dobra, nada a corrigir**. Só dividendos eram modelados.
**Caveat:** hoje a XConnect está sem excesso (saldo apertado) → dividendo modelado já era 0; a guarda fica
defensiva (ativa quando houver superávit). Resolve o item [ALTA] da auditoria de 14/jun.

## ✅ Buffer de caixa mínimo agora DINÂMICO (16/jun) — antes era 80k fixo chutado
O `_DIV_BUFFER_OP` (piso de caixa que trava a distribuição de dividendos: `caixa_minimo = impostos_est
+ ads_est + buffer`) era **R$ 80k FIXO hardcoded, sem derivação**. Agora `_buffer_opex_recorrente(rows,
now, meses=3, margem=0.15)`: média do **opex recorrente CONSOLIDADO** (as 2 empresas juntas — é um buffer
GERAL, não por empresa, igual à gerência/DRE) dos últimos **3 meses fechados** **+15%**, excluindo
impostos/ADS (já modelados) e INSUMOS/ICMS (CMV/tributo, não opex). Fallback 80k se <2 meses de histórico.
Exposto em `div_buffer_opex` (valor/media/detalhe) no `/api/cashflow`. **Hoje: R$ 56.705** (média 49.309
×1,15; Mar 53.764 / Abr 43.038 / Mai 51.125) vs 80k. Decisão CFO 16/jun: 3 meses, margem 15%, consolidado.

## ✅ IMPLEMENTADO (16/jun) — imposto do mês vigente não conta em dobro (casa por COMPETÊNCIA)
Antes, só lançamento PENDENTE suprimia a estimativa do mês corrente (`_tax_overrides`); imposto já
PAGO (efetivado, no saldo_real) → estimativa cheia subtraída 2×. Fix: o efetivado deste mês também
entra no override, **casando por COMPETÊNCIA** (campo COMPETENCIA do Sheets). **Crucial (alerta CFO):**
parcelamento de período antigo NÃO suprime a competência corrente — ex.: ICMS jan-abr pago em jun vai
p/ override de (2026,01..04), não toca a estimativa de maio; só DIFAL maio (competência corrente) reduz.
DIFAL segue residual; demais suprimem 100%. Loop sobre `all_rows_proj` (já filtrado por empresa).
**Provisão NÃO modela opex** (usa contas a pagar reais `get_contas_pagar_por_dia`) nem buffer → sem dobra de opex lá.
⚠️ CI (892a91e) falhou por E702 (`;`) nos helpers — corrigido (commit de style); sem mudança de comportamento.

## ⏳ PENDENTE (CFO 16/jun) — empréstimo ML: pagamento MÍNIMO de R$ 30.500/mês
O empréstimo sobre faturamento ML tem retenção de 25% do repasse, MAS existe um **pagamento mínimo
mensal de R$ 30.500**. Se os 25% de retenção não alcançarem esse valor no mês, precisa **cobrir a
diferença** (sai caixa extra). Hoje o modelo (cashflow `_loan_schedule` + provisão) provavelmente só
aplica os 25% (cap), sem o piso. **Revisar no GRÁFICO de fluxo de caixa E na PROVISÃO**: garantir que a
parcela do mês = max(25% do repasse, R$ 30.500). A pedido do CFO.

## ⏳ PENDENTE (CFO 16/jun) — revisar card "Havan a faturar (pedidos abertos)"
Card mostra R$ 832.080 (8 OCs, recebimento = entrega+~121d), − imposto ~121.690, − insumos ~63.517
(compra antes da entrega; reflete o abatimento dos insumos já lançados). Recebimentos: out/26 183.080,
nov/26 288.375, dez/26 360.625 — "a maioria além dos 60d desta provisão". Revisar a apresentação/uso
do card (recebíveis confirmados mas fora da janela curta da provisão; e o líquido pós-antecipação).

## ⏳ PENDENTE — adicionar CMV de reposição do ML na PROVISÃO (quando crescer)
A provisão não modela `cmv_base`/`cmv_imas`/`cmv_varejo88` (reposição de corpos 66mm + imãs + IM-88 do
ML) que o cashflow tem. Hoje é ~R$0 na janela (cmv_base só out/2026 = 19.453; imãs gated; IM-88 não
fechou), então DEFERIDO. Implementar quando a reposição ML voltar a ter valor (extrair helper do
`_base_payment_schedule` p/ os dois usarem). Dividendo (condicional) já foi adicionado à provisão (16/jun).

## Reconciliação gráfico × provisão (16/jun) — o que cada um modela
Só no GRÁFICO: dividendos (agora também na provisão), CMV reposição ML (deferido), empréstimo capital de
giro (simulação). Só na PROVISÃO: releases ML dia a dia (money_release real; o gráfico usa estimativa
mensal), disponível p/ antecipação ML hoje. Em AMBOS: Sicredi, Havan OCs (recebível+imposto+CMV insumos,
com antecipação), impostos, ADS, empréstimo ML (piso 30.5k), parcelamento, buffer dinâmico.

## 16/jun — provisão: dividendo condicional REVERTIDO + quitação do empréstimo com piso
- **Dividendo condicional na provisão: REVERTIDO.** Distribuía o teto (R$120k) no PICO temporário da
  antecipação de OCs (recebível trazido do futuro com desconto = financiamento, NÃO lucro distribuível)
  → saldo absurdo (-529k que o CFO viu). Dividendo fica só no gráfico; provisão = obrigações puras.
- **Quitação do empréstimo ML (card):** estimava só com 25% (dava jan/2027). Corrigido p/ acumular
  `max(retenção 25%, piso R$30.500)` por mês → quita ~nov/2026 (saldo 138.904 / ~30.500). Card a revisar:
  Contratado 214k, Total a pagar 246.028, Pago 107.123, Saldo devedor 138.904, Abatido no período 77.990.

## 16/jun — cabos pagos no vencimento do cartão ML (não à vista)
Cabos das OCs: pagamento modelado movido de `entrega−30d à vista` para o VENCIMENTO DO CARTÃO ML
(dia 23; compra após dia 10 vence dia 23 do mês SEGUINTE). Mais preciso — surfaceia o cabo da OC
de 27/06 (cartão 23/06) que antes era assumido pago 28/05 (passado, fora da projeção). Provisão 120d
c/ todas OCs antecipadas: 12/10 -77k→-110k (mais verdadeiro). Descasamento insumo×recebível é REAL
(paga no cartão dias antes do recebível da OC); a antecipação das OCs é o que reduz o buraco. `_havan_aberto_cmv_lancamentos`.

## ⏳ PENDENTE (CFO 16/jun) — fila de revisão do caixa
- **Provisão card + antecipação some ao trocar filtro:** mesmo com "antecipar todas as OCs" marcado,
  ao trocar o filtro p/ 120d a provisão mostra -556.172 em 12/10 (= número SEM antecipação; COM dá -167k).
  CFO suspeita que o crédito da antecipação NÃO está sendo passado quando troca o filtro. Investigar o
  frontend (ProvisaoCard `dias` vs o finance.antecipa_havan_aberto do parent) — pode não re-passar a
  seleção ao mudar `dias`, ou o dashboard precisa rebuild/refresh. ⚠️ revisitar os CARDS da provisão.
- **Pró-labore mal classificado:** está em `CATEGORIAS_ABAIXO_LINHA` (api.py:192) junto com dividendo.
  É despesa OPERACIONAL (salário do sócio), ≠ dividendo (abaixo da linha). get_overview soma de volta em
  despesas (api.py:1996-2005), mas a classificação base está errada → revisar no DRE com cuidado (não dobrar).
- **Reconciliação caixa×P&L (16/jun):** o "ML queima 82k/mês" foi ERRO meu — joguei CMV/insumos (192k =
  estoque), empréstimos pessoais (122k = financiamento), pró-labore (80k) e CAPEX no "burn". O ML é
  lucrativo (~20% rentab.). Caixa negativo = capital de giro (compra estoque) + amortização de dívida +
  pró-labore + ciclo Havan, NÃO prejuízo. Provisão é CAIXA, não P&L.

## 🔑 REGRA (CFO 16/jun) — TODA variável de simulação reflete no GRÁFICO E na PROVISÃO dia a dia
Princípio de design: qualquer parâmetro de simulação (parcelamento de impostos, antecipa_havan,
antecipa_havan_aberto, Sicredi, empréstimo giro, etc.) DEVE refletir igual no `/api/cashflow` (gráfico)
E no `/api/provisao` (dia a dia). Gating no frontend deve ser idêntico (ex.: antecipação por OC NÃO exige
financeMode — corrigido 16/jun). **Status:** finance_taxes ✅, antecipa_havan ✅, antecipa_havan_aberto ✅
(corrigido), Sicredi ✅. **FALTA: `emprestimo_giro` (capital de giro)** — está só no cashflow, não na
provisão (nem param nem lógica). Pendente implementar na provisão (PV entra hoje + parcelas saem/mês).

## ⏳ PENDENTE (CFO 17/jun) — revisar ritmo de vendas ML 7 dias (projeção do mês)
O cashflow projeta vender ~1046 cases no mês usando a métrica dos últimos 7 dias (7d×30/7). CFO suspeita
que está alto/não reflete a realidade. Revisar `_demand_combined_7d` / a projeção mensal de ML por ritmo
7d vs 30d — ver se o 7d está extrapolando um pico. Afeta cmv_base/pipeline e a estimativa de receita ML.
