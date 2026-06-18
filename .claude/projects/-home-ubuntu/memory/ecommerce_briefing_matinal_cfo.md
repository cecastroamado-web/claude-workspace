---
name: ecommerce-briefing-matinal-cfo
description: Plano (a implementar) de um briefing matinal único no Telegram pós-coleta 06h — o que um CFO precisa saber na 1ª hora
metadata: 
  node_type: memory
  type: project
  originSessionId: 5c7eb85e-fc5e-4fc1-80b9-ec8a21e0ed02
---

**✅ "Bom dia, CFO" GERAL IMPLEMENTADO (18/jun, commit `d350c0e`):** job `briefing-matinal` no
scheduler.py (`_run_briefing_matinal`/`_loop_briefing_matinal`), **06h30**, **pula domingo + feriado**
(`_FERIADOS_BR` hardcoded 2026-27, atualizar anual). Digest único, **suprime seção sem sinal**,
`_notify` bool → só marca `briefing_matinal_last` se enviou. **ADICIONAL** aos alertas existentes
(decisão CFO 18/jun). 5 seções (MVP; Quadro por item D-1 → fase 2):
1. **💰 Caixa hoje** — saldo/empresa (`get_bank_balance`) + total + posição do dia (entra−sai do
   dias[0] da provisão) + ⚠️ se `saldo_minimo_projetado` (provisão 7d) < SALDO_MINIMO (30k).
2. **📈 Vendas** — ontem (`get_ml_sales_day` + `get_external_sales_day`) vs média 7d + projeção do
   mês = MTD + avg7d×dias_restantes (`get_ml_sales_month_projection`).
3. **🏬 Havan** — em-trânsito (R$+qtd) + a faturar OCs (`_havan_enriched_snapshot`/provisão).
4. **🧾 Obrigações** — impostos (CATEGORIAS_TRIBUTO) + EMPRESTIMO A VENCER em 7d (Sheets).
5. **🩺 Saúde** — falhas de sync (scheduler_state).
Validado 18/jun: caixa (Av 16k/XC 4,8k, ⚠️ -96k em 23/06), vendas (ontem 23.952 ↓21%, proj 225k),
Havan (132.330/2.975 itens, a faturar 699.750). Obrigações+Saúde suprimidas (sem sinal).

---

**IMPLEMENTADO 11/jun/2026** (commit `7b26022` no ecommerce-agent): alerta Havan dedicado +
sobre-estoque por velocidade 7d + coluna vel 7d no card de produtos. Detalhes técnicos abaixo na
seção "ESCOPO DECIDIDO". O roadmap "Bom dia, CFO" GERAL segue pendente.

- **Alerta:** `build_havan_briefing_text(snap)` em `agent/api.py` (recebe snapshot enriquecido) +
  job `_run_havan_briefing` no `scheduler.py`, chamado no fim de `_run_havan_sync` (06h, 1×/dia,
  guard `havan_briefing_last_sent`). Bloco monoespaçado: por item vel 7d/dia ▲▼, proj mês, estoque,
  cob7, a faturar; totais un + R$ atacado; flags ⚠️cob baixa / 📦sobre-estoque / 💤sem giro.
- **Sobre-estoque** (`_havan_overstock`): `cobertura_alvo=4.0`, cobertura pela `_havan_velocidade_mensal`
  (7d, fallback 30d, senão 'sem_giro'); nível por cob7 (>6 alto, >4 médio); campos novos no item
  `cobertura_7d/janela_velocidade/venda_7d` + retorno `sem_giro/qtd_sem_giro`.
- **Frontend:** card produtosDetail ganhou 'Vel 7d/d' + 'Cob. 7d'; tabela sobreDetail usa cob7;
  tipos `HavanSobreEstoqueItem` em `lib/api.ts`.

**Ajustes 12/jun (commits `332311e`):**
- **Alerta ganhou coluna `nv`** = vendas novas desde a última coleta (Δ venda_30d entre as 2
  últimas capturas — MESMO número do card "O que mudou" do painel). Job injeta `venda_novas` por
  produto via `get_havan_last_two_captures`/`get_havan_capture_rows`. + total do dia no rodapé.
- **Painel: cobertura nas DUAS lentes** — `Cob. 7d` (velocidade recente, base do alerta) E
  `Cob. hist` (cobertura_meses 30d) lado a lado. Helper `cob7Prod()`/`fmtCob()` no index.tsx; nos
  cards de produtos e sobre-estoque (visíveis) + modais produtos/sobre-estoque/estoque/ruptura. O
  painel estava misturando as duas bases → parecia que o alerta "não batia".
- **Fuso captured_date (commit `a5baf24`)**: era UTC → coleta noturna virava captura do dia
  seguinte. Agora `date('now','-3 hours')` (BRT). Ver detalhe no corpo abaixo.
- **Coleta 06h pode dar TIMEOUT** (portal Havan lento): há retry 06→07→08h (`_loop_havan_sync`).
  Em 12/jun 06h deu timeout, 07h coletou OK — comportamento esperado, não é falha.

---

PLANEJADO em 11/jun/2026 (a pedido do CFO).

**ESCOPO DECIDIDO (11/jun): começar por um ALERTA DEDICADO SÓ DA HAVAN**, NOVO e SEPARADO dos
alertas existentes (não misturar no digest geral). Entrega: **mensagem Telegram (texto/tabela)**,
disparada logo após o `havan-sync` das 06h (a coleta que alimenta o snapshot Havan). As demais
seções do "Bom dia, CFO" abaixo ficam como roadmap futuro. O alerta Havan usa SÓ o snapshot Havan
(`havan_snapshot`) — campos por produto já disponíveis: `produto, sku, preco(varejo), preco_atacado,
vendas_mensais{MM/AAAA}, total_periodo, venda_7d, venda_30d, estoque, estoque_previsto,
pedidos_abertos, cobertura_meses, ruptura, recebido_mensais, receita_atacado_30d` + `em_transito_qtd`
(computado no enriquecimento). Conteúdo do alerta Havan (quadro por item + projeção):
- **Por item:** venda_7d (vel/dia) × venda_30d (vel/dia) c/ ▲▼; **MTD (vendas_mensais do mês)**;
  **projeção do mês = MTD + vel_7d × dias restantes**; estoque + cobertura_meses; ruptura;
  pedidos_abertos; **a faturar = (pedidos_abertos − em_transito) × atacado** (R$).
- **Totais:** MTD, projeção do mês, valor total a faturar.
- ⚠️ CAVEAT a alinhar: Havan NÃO tem "venda do dia anterior" granular (portal atualiza ~1×/dia,
  dá acumulados) → o pulso é venda_7d, não D-1. Projeção em UNIDADES (sell-out); definir se quer
  também R$ (atacado faturável vs varejo sell-out). Tabela larga (~90 chars) NÃO cabe no Telegram
  mobile → usar layout COMPACTO (2 linhas por item ou colunas reduzidas).
- Padrão técnico: novo job no `scheduler.py` (ex. `_run_havan_briefing`), legacy Markdown sem
  escapes, `_notify` bool → só marca scheduler_state se True, catch-up no restart.

**DECISÕES do CFO (11/jun):** (1) cobertura E sobre-estoque usam **velocidade 7d** (estoque ÷
(venda_7d/7×30); fallback venda_30d se venda_7d=0; venda 0 nas duas janelas = "sem giro", categoria
à parte, não sobre-estoque por excesso). (2) Projeção do mês em **unidades + R$ atacado faturável**
(qtd × preco_atacado). Layout do alerta: bloco monoespaçado compacto p/ caber no celular, ▲▼ de
aceleração, ⚠️ cobertura baixa / 📦 sobre-estoque / 💤 sem giro.

**PEDIDOS RELACIONADOS (11/jun) — mesma base de velocidade 7d:**
2. **Reformular SOBRE-ESTOQUE na Havan** (`_havan_overstock` api.py:8477): hoje a cobertura usa
   `venda_30d`/`cobertura_meses` (histórico) e ignora a aceleração recente → FALSO POSITIVO em item
   que esquentou nos últimos 7d. Caso real (snapshot 11/jun): **SUPORTE CASE PLÁSTICO C/ÍMÃ** marcado
   sobre-estoque (cob30=6,0m) mas vel 7d (8/d, ▲ vs 4,6/d) dá cob real **3,4m** → não é sobre-estoque.
   Correção: **cobertura efetiva = estoque ÷ (max(vel_7d, vel_30d) × 30)** (a janela mais rápida =
   menos cobertura = conservador contra falso positivo); e/ou **excluir do sobre-estoque quem acelera**
   (vel_7d > vel_30d). Sobre-estoque GENUÍNO hoje (alto nas 2 janelas): Cabo 12V 3M (~10m), Slim Pro
   (~17m), Suporte Fixo Haste (~11m). Afeta o card do dashboard E o futuro alerta. Cabos USB-C 5M e
   Painel Mini têm venda 0 → cobertura ∞, tratar à parte (item morto ≠ sobre-estoque por excesso).
3. **Card "Produtos — vendas, estoque e cobertura"** (dashboard Havan): adicionar coluna de
   **velocidade de venda 7d** (qtd/dia) ao lado de vendas/estoque/cobertura, p/ o CFO ver o ritmo
   recente vs a cobertura. Mesma fonte `venda_7d` do snapshot.

---

ROADMAP FUTURO — "Bom dia, CFO" GERAL (separado, depois): consolidar num único digest ~06h30 (após
coleta 06h: sync bancário Inter + havan-sync + NF-e Bling 06h30) o que importa na 1ª hora. Princípio:
**uma mensagem**, de cima (dinheiro hoje) p/ baixo (tendência); **suprime seção sem sinal**; cada
linha acionável. Crítico (saldo projetado < mínimo, atraso Havan grande, falha de coleta) vira push.

Hoje os alertas existem mas ESPALHADOS (saldo 10h, a-vencer 07h45, vendas 09h, ruptura no havan-sync
06h, etc.) — ver [[Alertas CFO ecommerce-agent — jobs do scheduler]]. A proposta NÃO remove os jobs;
agrega num digest matinal. ✅=já existe (reusar fonte)  🆕=novo.

**Seções propostas (ordem = prioridade do CFO):**
1. **CAIXA HOJE** — saldo consolidado/empresa (Inter) + Δ vs ontem ✅; aplicado em RF ✅; a liberar ML
   hoje (provisão) ✅; saldo MP a sacar ✅; títulos a pagar hoje+2d (valor+quem) ✅(check-avencer);
   **posição do dia: entra X − sai Y = sobra/falta Z** 🆕; ⚠️ se saldo projetado < mínimo em algum
   dos próx. 7 dias (fonte /api/cashflow D+90) 🆕.
2. **RECEBÍVEIS/RISCO** — Havan: a receber total + próximas entradas (entrega+120d), em-trânsito (a
   entregar), a faturar (fontes já no /api/havan) ✅; atrasados se houver 🆕; divergência de
   conciliação que peça ação 🆕.
3. **VENDAS (pulso)** — faturamento ontem (ML+Bling+NFS-e) vs média 7d/30d % ✅; mês corrente
   run-rate vs mês anterior/meta 🆕; nº pedidos/ticket ✅. **Projeção de fechamento do mês pela
   velocidade dos últimos 7 dias** 🆕 (pedido do CFO, explícito): mostrar **(a) quanto já vendeu
   no mês (MTD, R$ e qtd)** + **(b) projeção do mês = MTD + velocidade 7d (R$/dia e qtd/dia) ×
   dias restantes**, total e por item; comparar c/ mês anterior e % de evolução. Mesma janela 7d
   do quadro por item.
3b. **QUADRO POR ITEM (D-1)** 🆕 — peça central pedida pelo CFO (campo a campo). UMA linha por
   SKU, com: **vendas do dia anterior (qtd + R$)** · **média 7d × média 30d (qtd/dia) c/ seta ▲▼
   de aceleração/desaceleração** (item esquentando 7d≫30d / esfriando 7d≪30d) · **estoque atual** ·
   **ruptura (status: zerado/negativo/dias de cobertura)** · **pedidos em aberto** · **valor a
   faturar** (= pedidos abertos − em-trânsito × atacado, p/ Havan). Visão UNIFICADA ML + Havan +
   B2B. Ordenar por relevância (giro × R$); destacar o que não vendeu nada vs média e o que está
   em ruptura. Fontes já existentes: `ml_orders_sync` + itens Bling (vendas D-1), médias como a
   sugestão de pedido de imãs ([[ecommerce-ima-sugestao-pedido]]), snapshot Havan
   (pedidos_abertos/em_transito/a-faturar — ver [[ecommerce-havan-backlog-faturamento-transito]]),
   estoque por item. Provavelmente exige consolidar SKUs ML×Havan num catálogo comum.
4. **OPERAÇÃO/RUPTURA** — ruptura ML (parados/sem estoque) ✅; ruptura Havan na ponta ✅; runway de
   imãs + "pedir agora?" considerando lead time ✅(sugestão de pedido); catálogo ML pausado ✅.
   (Parte da ruptura já fica visível no Quadro por item acima; aqui é a visão consolidada/acionável.)
5. **OBRIGAÇÕES/IMPOSTOS** — impostos a recolher na semana (DAS/DARF/ICMS/DIFAL) c/ vencimento 🆕;
   parcelas de empréstimo do mês (Sicredi/ML) 🆕.
6. **SAÚDE DE DADOS** — falha de sync (NF-e integrity ✅, token Drive ✅, portal Havan ✅);
   anomalia de pedidos ✅.

**Decisões pendentes do CFO antes de codar:** (a) digest único vs manter separado + só adicionar os
🆕; (b) quais seções entram no MVP; (c) horário exato (06h30 após NF-e?) e se domingo/feriado roda;
(d) limiar de saldo mínimo p/ o ⚠️ de caixa (hoje SALDO_MINIMO=30k no check-saldo).

Padrão técnico ao implementar: legacy Markdown (sem escapes \), `_notify` retorna bool → só marca
state se True (ver regra em CLAUDE.md/[[ecommerce-alerta-enviado-falso]]); catch-up no restart;
scheduler_state p/ não duplicar. Relacionado: [[ecommerce-provisao-pontos-cegos]],
[[ecommerce-money-release-date]], [[ecommerce-havan-backlog-faturamento-transito]].
