---
name: ecommerce-rentabilidade-padronizacao
description: "Padronização da rentabilidade Havan entre telas — Mark 1% no engine canônico, cabos Simples sem crédito ICMS, painel custo, filtro 587"
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

Auditoria CFO (18/jun/2026): "não pode um lugar estar certo e outro não" — telas de rentabilidade divergiam. Engine canônico = `compute_profitability` (metrics.py); painel de canal `_havan_channel_margin` (api.py:9291) é motor SEPARADO.

## ✅ Feito nesta rodada (no ar, 48 testes ok)
- **NF 587 cancelada** vazava no detalhe do cliente: `/api/customer-orders` (api.py:~17625) não filtrava `exclude_from_revenue` (NF-e E NFS-e). Corrigido.
- **NF 586 painel CMV=0**: SKU Havan `HVNSPPASLMIVE4C` (nome sufixado `#HVN#`) não casava no engine (só `cost_override` resolvia, e o painel não tinha). Fix: entrada em `nfe_product_cost_map` → `Case Painel Mini c/ Ventosa` (custo R$41,11 = Produto 35,01 + Ventosa 6,10) + caixa via `havan_embalagem` (já existia). CMV real = R$42,88/un.
- **Cabos fornecedor Simples (jun+)**: crédito ICMS 0% desde 01/06/2026 p/ TODOS os cabos (12v 2m/5m, USB-C 2m/3m/5m; 12v 3m já estava). Linha nova em `product_costs` (effective_from='2026-06-01', icms_credit_rate=0, custo mantido). Date-aware: maio continua 4%. Muitos `#HVN#` mapeiam p/ "Cabo 12v 3m" no `nfe_product_cost_map`.
- **Comissão Mark 1%** entra no engine: param novo `havan_mark_rate` em `compute_profitability`; gate `"HAVAN" in customer_name`; deduzida ANTES da comissão do vendedor. Passado em api.py:7363 (`_HAVAN_COMISSAO_MARK_RATE=0.01`). Coluna `comissao_mark` no output + somada em `fees` do customer-ranking + linha no modal de rentabilidade/index.tsx.
- **Rafael 15% s/ lucro** JÁ era aplicado via `auto_rates` (api.py:7495-7509), DEPOIS do Mark — consistente. Não faltava.
- **Carry 121d**: decisão CFO = NÃO deduzir (fica só na simulação).
- **🐛 BUG zero-falsy no crédito ICMS (crítico)**: `compute_profitability` (metrics.py ~351) e `/api/order-detail` (api.py ~7967) resolviam a alíquota de crédito com cadeia `A or B or ... or 0.17`. Taxa legítima **0.0** (cabo Simples) é falsy → caía no **default 17%**. Resultado: TODAS as NFs de junho com cabos creditavam 17% indevido (NF 589, 1000× cabo 12v 3m: crédito fantasma R$3.906→0). Corrigido com checagem explícita de `is None`. NF 589 detalhe foi de 21.900,93/39,1% → **18.104,32/32,3%**. Agregado Havan jun: 26,3% → **23,5%** (real).
- **`/api/order-detail`**: era 3º motor de P&L (alimenta detalhe por NF em "Por Cliente" E modal "Por Pedido"). Agora aplica Mark + crédito canônico. Antes divergia da rentabilidade por pedido.

Validado live jun/2026 XConnect Havan: 6 NFs, receita 418k, lucro 110k, **margem 26,3%** (antes inflada por painel sem custo + Mark ausente).

## ✅ #11 FEITO (18/jun) — caixa + Mark nas telas secundárias
- **Canal Havan** (`_havan_channel_margin`): adiciona caixa Metagraf (cmv_delta/credit_delta) por SKU, date-aware + match por prefixo; `_havan_unit_cost` já era correto no crédito (multiplica direto, sem bug or). PDF (`havan_report.py`) herda. Painel canal agora bate CMV com engine (Case 178,86 / Painel 42,88 / Slim 62,42 / Haste 49,58).
- **commission-report** + **vendor-roi**: caixa no CMV (passe separado por SKU) + Mark deduzido antes dos 15% do Rafael. Decisão CFO: base do Rafael inclui caixa+Mark (comissão dele cai um pouco). Batem com o engine (NF 589 Rafael 3.194,88; 586 2.886,28; 582 3.617,93). Também corrigido o **bug zero-falsy do or-rate** nesses dois.
- **BUG do conn fechado** no commission-report: `_load_havan_box_map(conn)` rodava após `conn.close()` → mapa vazio. Movido p/ antes do close.
- **#14**: canal mantém ICMS débito flat 12% (taxa interestadual modelada p/ SC — aceitável; é visão por produto/velocidade, não por NF). Não unificado de propósito.

## ✅ Simulação + Slim Pro 2 (18/jun) — FECHADO
- **Slim Pro 2** (`HVNCSSLVICU00Q2`): caixa Metagraf cadastrada em `havan_embalagem` = mesmo custo do Slim Pro (emb 3,28 + IPI 0,492, desde 2000-01-01).
- **Simulação** (`/api/havan/simulacao-pedido` + `simulacao-havan/index.tsx`): caixa adicionada ao `cmv_vigente`/`cred_vigente` (mesma lógica do canal). Já tinha Mark+Rafael+ICMS 12%. Toggle "Cabos no custo novo" → renomeado "Simular custo flat dos cabos" (o 0-crédito já é vigente; o toggle só troca o CMV pelo flat). Validado: Case 178,86 / Slim 62,42 / Slim Pro 2 62,42 / Painel 42,88 / Haste 49,58; cabos crédito 0.
- Kits de cabo (Kit X Cabos): NÃO vendendo agora → crédito não zerado de propósito (CFO).

**AUDITORIA DE RENTABILIDADE 100% CONCLUÍDA** — todas as telas (pedido/cliente/order-detail/canal/PDF/commission/vendor-roi/simulação) consistentes.

## ✅ order-detail caixa (18/jun, pós-commit 98eb645) — última divergência fechada
order-detail (detalhe da NF no modal) calculava comissão do vendedor SEM a caixa no `_cmv_d` → divergia do commission-report (NF 582: 3.944,37 vs 3.617,93). Fix: passe da caixa Metagraf por SKU no `_cmv_d` (date-aware, prefixo), antes do P&L. Validado 582/586/589 batem order-detail = commission-report. **Commitado** (submódulo 498eef7 / pai d264cb3).

Ver [[ecommerce-havan-dashboard]], [[ecommerce-icms-aliquotas-reais]], [[ecommerce-havan-liquido-carry-sku]], [[ecommerce-havan-embalagem-metagraf]].
