---
name: ecommerce-havan-dashboard
description: "Dashboard do canal Havan (ecommerce-agent) — scraper, frequência 7h+18h, card de reposição 120d, conciliação"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8754b4d1-8965-4368-9264-3cf095793a05
---

Dashboard dedicado do canal Havan no `ecommerce-agent` (`agent/havan_scraper.py`, `/api/havan`, `dashboard-ui/src/features/havan/index.tsx`). Scrape do Portal do Fornecedor Havan via Playwright (endpoint interno `POST /Relatorios/ObterDados`, varre mês a mês desde jan/2026; credenciais `HAVAN_USER`/`HAVAN_PASS`).

**Frequência do scraper (jun/2026):** roda **2×/dia, 7h e 18h** (`_loop_havan_sync` → `_wait_until_any_hour([7, 18])` em `scheduler.py`). **Why:** 1×/dia já basta para a decisão de reposição (lenta, cobre 165d), mas a rodada das 18h mantém o dashboard fresco capturando o sell-out do dia. **How to apply:** o **alerta de ruptura no Telegram dispara só na rodada da manhã** (`_run_havan_sync(alert_rupturas=datetime.now().hour < 12)`); a das 18h só atualiza o snapshot sem re-notificar (evita alerta duplicado). NÃO reverter para 1×/dia nem ligar alerta no run noturno sem o usuário pedir.

**Card "Sugestão de reposição — próximos 120 dias"** (`reposicao_120` em `/api/havan`, helper `_havan_reposicao_120` em `api.py`): modelo HÍBRIDO decidido com o usuário —
- base diária = blend(60% `venda_30d` + 40% média dos meses cheios)
- ajuste de tendência limitado a ±30%
- estoque de segurança = z·σ·√(lead+120), **nível de serviço 95% (z=1,65)**
- **lead time 45d**, cobre 165d; desconta `estoque` + `pedidos_abertos` (já no canal/a caminho)
- mês corrente é parcial → sempre excluído do cálculo
- score de confiança por produto (nº meses c/ venda × CV)
- Observação: a Havan costuma ter `pedidos_abertos` enormes → maioria dos itens dá repor=0 (legítimo, coberto). O card mostra estoque+pedidos pra explicar os zeros.

**Conciliação** (`snap["conciliacao"]`): nosso faturamento Bling NF-e→Havan (`situacao=6`, desde 2026-01-01, exclui `exclude_from_revenue`) × valor que a Havan registrou receber (a atacado) → divergência.

**Preço de atacado XConnect→Havan** (≠ preço varejo Havan): vem dos **itens REAIS da NF** — `set_havan_snapshot` sobrepõe `preco_atacado`/`receita_atacado_30d` com o último `unit_price` de `order_items_detail` por SKU (match exato + prefixo); é a mesma fonte que a página de rentabilidade usa, e auto-atualiza quando a Havan renegocia. Mapa keyword `PRECO_ATACADO` no scraper é só **fallback** (p/ SKU sem NF, ex.: USB-C 5M, Painel). Preços validados vs NF (jun/2026): Case 295, 12V **2M=54** (subiu de 50; era falso positivo de R$2k na conciliação de maio), 12V 3M 56, Slim 85, USB-C 3M 50, Haste 95.

**Evolução jun/2026 — 5 features (roadmap "fazer tudo"):**
1. **Histórico de snapshots** (fundação): tabela append-only `havan_product_history` (1 linha/produto por coleta 7h/18h), gravada em `set_havan_snapshot`. `GET /api/havan/history?days=N` → série + "o que mudou" (diff das 2 últimas coletas). Habilita forecast com acurácia/backtest.
2. **Receita Havan no fluxo de caixa** (`/api/cashflow`): campo `receita_havan_est` por mês + `havan_proj_total`. Conceito alinhado com CFO: "A Receber" já cobre NFs emitidas; projeta SÓ recompra futura (sell-out × atacado), começando após esgotar cobertura (estoque+pedidos), caixa em **entrega + 120d** (régua: faturamento → +15d agendamento CD → +120d). NÃO duplica A Receber.
⚠️ **CMV do Case Havan = R$176,28 (88mm), NÃO 114,16 (66mm).** O Case da Havan é case 66mm montado com **imãs 88mm** (override component 121 = 4×R$31,76 = R$127,04; cost_override real da NF = R$176,28). A margem Havan lê o `cost_override` de `order_items_detail` (mesma fonte da aba rentabilidade) e mapeia o Case → "Case Injetado 88mm XConnect". Ver [[ecommerce-nfe-discount-override-descontos-contratuais-por-nf]]. **SEMPRE conferir overrides de custo (order_item_component_override / cost_override) antes de atribuir CMV de produto Havan.**

**Verificação completa preço × custo (jun/2026) — tudo conferido vs NFs reais:**
- **Overrides de custo:** só o **Case** diverge (176,28 vs 114,16 genérico) — corrigido. `order_item_component_override` tem 1 registro (Case); `cost_override` em `order_items_detail` em 2 SKUs: Case (176,28) e 12V 2M (38,16 = custo padrão, sem impacto). Demais produtos sem override.
- **Preços de atacado:** todos batem com a última NF (`unit_price` de `order_items_detail`) — Case 295, Haste 95, Slim 85, 12V 3M 56, 12V 2M 54, USB-C 3M 50. Único que renegociou foi o **12V 2M (50→54)**, já tratado. USB-C 5M/Painel nunca vendidos à Havan (preço do fallback, 0 vendas).
- Mecanismo: preço (unit_price) e custo (cost_override) vêm de `order_items_detail` automaticamente → renegociações/overrides futuros entram sozinhos, sem mexer no código. Resultado líquido do canal após correção: **~16%** (não 24%).

3. **Margem + P&L do canal** (`/api/havan.margem`): cascata completa, **mesmas fontes do painel de rentabilidade** (`compute_profitability`). Resultado líquido = receita atacado − **desconto 2%** (HAVAN 1% comercial + 1% logística, `customer_discount_rules`) − **CMV** (`product_costs` via `_HAVAN_CMV_MAP`) − **ICMS líquido** (déb 12% interestadual SC, validado vs NFs reais CFOP 6102, − crédito insumos) − **PIS/COFINS 3,65%** − **IRPJ/CSLL 2,28%** (LP) − **frete** (custo_frete real das NFs, **rateado POR NOTA, dentro da nota POR VALOR de cada item** (qtd×preço) — decisão do usuário jun/2026: mais justo que por unidade, item caro puxa mais frete, cabos baratos não sobrecarregam; manter por valor, NÃO voltar p/ por unidade). Helper `_havan_freight_per_unit` casa cada NF aos itens via `order_items_detail` por valor total; NF de produto único = frete/un exato; junho em trânsito sem frete fica de fora. Frete/un: Case 5,96 · Haste 3,08 · Slim 3,47 · 12V 3M 2,05 · USB-C 3M 1,90 · 12V 2M 1,15) − **comissão Rafael 15%** sobre o lucro (frete entra ANTES da comissão; rate em `vendor_commission_config`). Markup Havan (varejo÷atacado). **Resultado real ~23% líquido** (R$13,5k/mês) — não confundir com a margem bruta 47%. Campos: `resultado_30d`/`resultado_pct`, `desconto/icms/pis_cofins/irpj_csll/frete/comissao_30d`.
4. **Sobre-estoque na Havan** (`/api/havan.sobre_estoque`; reframado jun/2026 — era "risco de devolução", premissa errada): cobertura alta × sell-through baixo (recebido≫vendido) → Havan comprou demais → **não recompra tão cedo** (receita de recompra pausa + capital parado). **NÃO é devolução** — a Havan não devolve; NF 547/552 foram reemissão por regime (ver [[ecommerce-havan-devolucoes]]). Sinal válido (entrega jan/2026 real). Campos: `flagged`, `total_valor_excedente`, `nivel`, `valor_excedente`. Alerta Telegram só de manhã. Case = ALTO (cob 7,4m, sell-through 32%, ~R$135k excedente parado).
5. **Conciliação mês a mês** (`conciliacao.por_mes` + `em_transito`): nossa NF × recebido Havan por mês; "não-entrada" = faturado − recebido (clock dos 120d não iniciou). Jun/2026: R$286k faturado, R$0 recebido → R$288k em trânsito.

**UX (jun/2026):** todos os cards e KPIs da página Havan são **clicáveis → abrem modal de detalhe** (Dialog; KPIs mostram breakdown por produto, cards mostram tabela completa + metodologia; ícone/hover destaca). O card **Margem & P&L tem filtro de período** no header (Run-rate 30d default / Todos / Ano / cada mês): a economia por unidade é run-rate (não muda), só o VOLUME muda (`vendas_mensais` do período × economia/un), recalculado no frontend. Seletores internos usam `stopPropagation` p/ não abrir o modal.

Régua de pagamento Havan (cravada com CFO): **120 dias da ENTREGA no CD Barra Velha** (não da emissão; há delay de agendamento ~15d default ajustável). Prazo confirmado: NFs Havan já emitidas estão em "A Receber" do fluxo.

Relacionado: [[ecommerce-ima-sugestao-pedido]] (demanda ML+Havan no pedido de imãs), [[ecommerce-havan-devolucoes]], [[ecommerce-havan-backlog-faturamento-transito]].
