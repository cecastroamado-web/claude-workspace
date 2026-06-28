---
name: ecommerce-havan-embalagem-metagraf
description: Custo da caixa Metagraf aplicado SÓ ao canal Havan (SKU HVN*) via hook no compute_profitability
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Caixa Metagraf — custo só no canal Havan (jun/2026)

A Havan recebe **caixa de papelão separada** (fornecedor **Metagraf**, NÃO "Metragraf"); o **ML continua com a caixa da Gedeval embutida na base** (corpo). Como os mesmos produtos vendem nos dois canais e `product_costs` é **por `product_name`** (SKUs compartilham os componentes — `_load_product_components_history` agrupa por product_name, api.py:~1363), NÃO dá pra ter custo diferente por canal direto no product_costs.

**Discriminador de canal:** Havan usa SKU **`HVN*`** em 100% dos itens; ML usa `MLB*`.

## Implementação (Opção A — hook por SKU HVN, escolhida pelo CFO)
- Tabela **`havan_embalagem(sku, produto, embalagem, ipi, base_delta, emb_icms_rate=0.12, base_icms_rate=0.17, desde)`** (db.py SCHEMA).
- Loader `_load_havan_box_map(conn)` + wrapper `_havan_box_map()` (api.py) → `{SKU_upper: {cmv_delta, credit_delta, desde}}`. `cmv_delta = embalagem + ipi − base_delta`; `credit_delta = emb×0.12 − base_delta×0.17`.
- Hook no **`compute_profitability`** (metrics.py, param `havan_box_map`): por item, se `sku.upper()` está no mapa **e** `order_date >= desde` → `cmv += cmv_delta×qty` e `credito_icms += credit_delta×qty`. **Aditivo e blindado**: ML (MLB) nunca casa; data-aware via `desde`. Passado nos **4 call sites** do compute_profitability (rentabilidade, alertas, ai, vendor-roi).
- **Base −2**: a base/corpo (Gedeval) já trazia ~R$2 de caixa; ao assumir a Metagraf separada, abate 2 da base (estorna o crédito da base nesse valor a 17%).
- **ICMS destacado real = 12%** nas NFs Metagraf (1.200,48/10.004 etc.), apesar da alíquota listar 17 — decisão do CFO: usar o destacado (12%). IPI 15% é **custo** (não credita p/ revenda).

## Config semeada (jun/2026)
| SKU Havan | Produto | embalagem | ipi | base−2 | desde |
|-----------|---------|-----------|-----|--------|-------|
| HVNCSPLIM660 | Case Injetado (case 66mm + imã 88 via override) | 3,98 | 0,597 | sim | todas |
| HVNCSSLPRVI00SC | Slim Pro | 3,28 | 0,492 | sim | todas (650+850=1500 pç) |
| HVNHSHAXXX008A | Haste | 2,58 | 0,387 | sim | 2026-06-01 |
| HVNSPPASLMIVE4C | Case Painel | 3,28 | 0,492 | sim | 2026-06-01 (500 pç faturadas) |

Custo unitário das caixas vem das **NFs Metagraf** (em `/home/ubuntu/nfs_entrada/`), todas X Connect matriz RS, entrega Gedeval, IPI 15%:
- **Slim → NF 162916/3, emissão 19/03/2026** — "ESTOJO CONNECT 245×120×45mm" (cód. 61612), **1.600 un × R$ 3,28 = R$ 5.248,00** (IPI 15% = R$0,492/un). Bate exatamente com `havan_embalagem` Slim Pro (3,28 + 0,492). ✅ CONFIRMADO 24/jun.
- Injetado/Mini → NF 163851/3 (17/04/2026) — "Cartucho Case Starlink Mini 336×332×52mm", 3.025×3,98.
- Haste → NF 165132/3 (29/05/2026) — "Cart Haste 250×100×100mm", 3.300×2,58.
- Painel → NF 165647/3 (15/06/2026) — "Cartucho Case Painel 380×180×50mm", 3.050×3,28.

## Mapeamento (importante — não confundir produto)
Itens Havan mapeiam via `nfe_product_cost_map`: Case `HVNCSPLIM660` → **"Case Injetado 66mm XConnect"** (+ `order_item_component_override` do imã 88mm = 127,04), NÃO o "88mm". Slim → "Case Slim Pro para vidros retos"; Haste → "Haste"; Painel → "Case Painel Mini c/ Ventosa".

## Slim Pro vs Slim Pro 2 (não confundir) — RESOLVIDO 27/jun/2026
- **Slim Pro** mantém **3 ventosas × R$ 11,57** (R$ 34,71). NÃO mexer. Na Havan = SKU `HVNCSSLPRVI00SC` → mapeia "Case Slim Pro para vidros retos" (CMV base ~R$ 60,64; com caixa ~R$ 62,42). Deixar assim.
- **Slim Pro 2** usa **4 ventosas baratas × R$ 3,05 = R$ 12,20** (vs 3×11,57 do regular). Corpo 25,935 + ventosas 12,20 = **base R$ 38,13**; com caixa Slim ~R$ 39,91. Diferença vs Slim Pro = exatamente **R$ 22,51** (34,71 − 12,20).
- **SKU novo da Havan JÁ EXISTE**: `HVNCSSLVICU00Q2` ("SUPORTE SLIM PRO 2 PARA STARLINK XCONNECT") no snapshot. product_cost dedicado **"Case Slim Pro 2"** (XConnect+Aviation, effective 2026-06-23: corpo 25,935@0.17 + ventosas 12,20@0.0389). `havan_embalagem` já tem o SKU com a caixa Slim (3,28+IPI, base−2).
- **BUG corrigido (api.py `_HAVAN_CMV_MAP`)**: a regra genérica `["SLIM"]` casava com Slim Pro **E** Slim Pro 2 → os dois pegavam o CMV do Slim regular (no painel `/api/havan` apareciam iguais). Fix: regra específica **`["SLIM","PRO 2"] → "Case Slim Pro 2"` ANTES** da `["SLIM"]` (1ª regra que casa vence). "PRO 2" distingue limpo (regular não contém). NÃO mexeu no HVNCSSLPRVI00SC. Validado no painel: Slim Pro 2 R$ 39,91 vs Slim Pro R$ 62,42.
- **Pendente quando houver venda real**: o preço de atacado/varejo dos dois está igual no snapshot (R$ 85 / R$ 149,90) — decisão comercial, não bug. E ao faturar NF-e real do Slim Pro 2, garantir o `nfe_product_cost_map` (nome da NF → "Case Slim Pro 2") p/ a rentabilidade por pedido. ICMS das ventosas (0.0389) provisório — revisar c/ nota do fornecedor.
- Ventosa transparente = insumo novo (`VEN-TRANSP` na planilha), compra de 9 mil prevista p/ pedido Havan. **Crédito ICMS 3,89% CONFIRMADO (27/jun)**: CFO recebeu a NF — fornecedor é do Simples mas a NF **autoriza aproveitamento de 3,89%**. Já gravado: Slim Pro 2 XConnect ventosas `cred 0,0389` (vai p/ Havan via `_havan_unit_cost` company='XConnect'). Aviation fica 0 (Simples não credita ICMS no modelo de rentabilidade). Ver [[ecommerce-ventosa-utimex-credito-simples]]. NÃO é mais provisório.

## Planilha de insumos (entradas FEITAS 17/jun; saídas pendentes)
Planilha de imãs/insumos = **.xlsx no Drive** `1a0G6lDQjMc...` (NÃO é Sheet nativo → Sheets API recusa; só Drive API download+openpyxl+`files().update`). Abas: Parametros/Entradas 2026/Saídas 2026/Estoque Atual 2026/Consumo Padrão.
- **Escrita**: `token_drive.json` foi re-OAuth p/ escopo **`drive`** (era `drive.readonly`, rebaixado 21/mai) via `scripts/setup_drive_oauth_write.py` (2 fases, não-interativo). Backup do token readonly em `token_drive.json.bak-*`. **Agora dá pra escrever na planilha.**
- **FEITO 17/jun** (com backup Drive "BACKUP Imas-Insumos pre-caixas"): Parametros linhas 7-11 (CX-SLIM, CX-INJ, CX-HASTE, CX-PAINEL, VEN-TRANSP); Entradas 2026 linhas 10-13 (4 compras Metagraf, Custo Unit = **item + IPI**: 3,772/4,577/2,967/3,772; entrega GEDEVAL); Estoque Atual fórmulas estendidas p/ as 5 linhas novas (Translator). Fórmulas antigas preservadas no round-trip openpyxl.
- **Gotcha fórmula (corrigido 18/jun)**: as colunas Descrição (B) e Mínimo (G) do Estoque Atual usavam `VLOOKUP(...,Parametros!$A$2:$E$6,...)` com **range fixo até a linha 6** → SKUs novos (linhas 7-11) não puxavam Descrição/Mínimo. Fix: estendi p/ `$E$1000`/`$B$1000` nas linhas 2-11. **Ao adicionar insumos, conferir ranges fixos de VLOOKUP.**
- **Saídas lançadas (17-18/jun)**: Slim (650+850, CX-SLIM) e Case (740+260, CX-INJ) o CFO marcou (já tinham linhas de produção com VEN-PRO/imã). **Haste** só NF 583 (500, CX-HASTE) — box-only, terceirizada, as anteriores não usaram caixa. **Painel** 500 (VEN-TRANSP 2/unid + CX-PAINEL), box-only-ish. Saldos: CX-SLIM 100, CX-INJ 2.025, CX-HASTE 2.800, CX-PAINEL 2.550, **VEN-TRANSP −1.000** (negativo até lançar a compra das 9 mil). Estoque mínimo: CX-* 1.000, VEN-TRANSP 3.000.
- **Baixa automática de caixas (FEITO 17/jun)**: aba Saídas ganhou coluna **N "Caixa (SKU)"** com dropdown (CX-SLIM/INJ/HASTE/PAINEL). A fórmula de Saídas no Estoque Atual (linhas 2-11) virou `SUMIFS(QntInsumos por SKU em D/G) + SUMIFS(QtdProduto col J onde Caixa col N = SKU)`. CFO marca a caixa nas linhas dos pedidos embalados → estoque da caixa baixa sozinho (1 por produto). Resolve as saídas retroativas sem eu calcular (CFO marca só as linhas reais, evita a bagunça de NFs canceladas 547/552/587). VEN-TRANSP sem entrada ainda (compra 9 mil futura).
- **Custo Unit da planilha = item + IPI** (decisão CFO); no `product_costs`/DRE o crédito de ICMS 12% é tratado à parte (não confundir as duas convenções).

## Pendentes da planilha de insumos (18/jun — encerrado por hoje)
- **✅ Alerta Telegram de estoque < mínimo (FEITO 18/jun)**: job `check-insumo-minimo` (scheduler.py, 08h30 diário). Lê a planilha (token_drive, openpyxl), computa saldo por SKU = `inicial (Fechamento+Saldos Iniciais) + entradas (Entradas col G por SKU col D) − saídas insumo (J×K por SKU col D) − caixas (J onde col N=SKU)` vs mínimo (Parametros col D). Alerta **só quando um SKU NOVO cruza abaixo** (anti-spam via scheduler_state `insumo_minimo_alerted`; recuperações sincronizam sem re-alertar). Computa em Python (NÃO confia no Estoque Atual calculado — `data_only` fica None após escrita openpyxl). Validado: abaixo do mínimo hoje = IM-66-M (0/2000), CX-SLIM (100/1000), VEN-TRANSP (2000/3000).
- **VEN-TRANSP saldo anterior**: tinha estoque antes; lançar na aba **"Saldos Iniciais 2026"** (linha 7: col A=VEN-TRANSP, col D=Quantidade Inicial). Hoje está −1.000 (consumo do Painel sem o saldo inicial). CFO vai passar a qtd.
- **✅ Slim Pro 2 / Painel auto-puxar ventosas (FEITO 18/jun)**: a col L "Produtos por unid (padrão)" da Saídas faz `VLOOKUP($C,'Consumo Padrão'!$A$2:$B$200,2)` por NOME (col C). Adicionei ao **Consumo Padrão**: **"Slim Pro 2" = 4** e **"Case Painel" = 2** (além de Supercase Mini=4, Slim Pro=3). Uso na Saídas: Produto=nome exato + SKU=VEN-TRANSP + Qtd Produto + **K vazio** → col G = J×L (4 ou 2). Slim Pro 2 não está no product_costs de estoque (nome "Slim Pro 2"). Caixa (col N): Slim Pro 2 usa CX-SLIM por enquanto (embalagem nova a definir).

Relacionado: [[ecommerce-bling-rules]], [[ecommerce-havan-dashboard]], [[ecommerce-havan-faturadas-antecipacao]].
