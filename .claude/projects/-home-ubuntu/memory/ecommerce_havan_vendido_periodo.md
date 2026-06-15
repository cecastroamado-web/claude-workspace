---
name: ecommerce-havan-vendido-periodo
description: "Havan — semântica dos campos de venda (venda_30d janela móvel vs total_periodo acumulado vs venda_7d) e o card \"O que mudou\" com coluna Vendido (período)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 26b35212-7516-47b6-a3d6-33af5680b921
---

Documentação detalhada para **não confundir** os números de venda do painel Havan (ecommerce-agent). Origem: 13/jun/2026 o CFO viu o case c/imãs com **"+17"** no card "O que mudou desde a última coleta" e estranhou — não tinham vendido 17 de uma coleta pra outra; em outro lugar via **9**. Os dois números estão certos, medem coisas diferentes.

## Os 3 campos de venda — o que cada um É e DE ONDE vem

Coleta automática 1×/dia ~06h (`havan_scraper.py`), uma linha por produto por coleta na tabela `havan_product_history` (chave `captured_at`).

1. **`venda_30d`** — campo **`qtdVenda30Dias`** do próprio portal Havan, do mês mais recente consultado. (`havan_scraper.py:172` agrega, `:198` vira snapshot, `:252` grava.)
   - É uma **JANELA MÓVEL de 30 dias** que o portal recalcula sozinho a cada scrape: vendas em [hoje−30, hoje].
   - **NÃO é venda diária nem venda entre coletas.** É indicador de **ritmo/aceleração**.

2. **`total_periodo`** — soma de `vendas_mensais` (mês a mês desde jan/2026). (`havan_scraper.py:248`.)
   - É o **ACUMULADO / sell-out real** do período inteiro.
   - O **Δ de `total_periodo` entre duas coletas = unidades realmente vendidas** nesse intervalo. ESTE é o número honesto de "vendeu X desde a última coleta".

3. **`venda_7d`** — consulta extra [hoje−7, hoje] somando `qtdVendasTotais`. (`havan_scraper.py:206-223,251`.)
   - Janela móvel de 7 dias; usada para ritmo recente (mensalizada ×30/7 em `havan_report.py`).

Outros do scraper: `recebido_periodo` (acumulado recebido), `estoque`/`estoque_loja`/`estoque_deposito`/`estoque_cd`/`estoque_transito` (são **foto de agora**, ruidosos dia a dia — não derivar venda de Δestoque), `pedidos_abertos`.

## Por que Δvenda_30d ≠ venda real (a matemática)

A cada dia a janela de 30d **soma a venda do dia novo e descarta a venda do dia que saiu por trás** (~31 dias atrás):

> Δvenda_30d = (venda do dia) − (venda do dia que saiu da janela)

Só coincide com a venda real se a venda for constante. Como o case está **acelerando** (dias antigos fracos saindo da janela), Δvenda_30d **infla** acima da venda real. Por isso aparecia "+17/+10" enquanto a venda real era +9.

## Exemplo real (case SUPORTE CASE PLASTICO C/IMAS, 2 últimas coletas)

| coleta | venda_30d | total_periodo (acum.) | venda_7d | estoque |
|---|---|---|---|---|
| 12/jun 10:00 | 152 | 408 | 63 | 810 |
| 13/jun 09:00 | 162 | **417** | 67 | 777 |

- **Venda real entre coletas = Δtotal_periodo = 417−408 = +9** ← o "9" correto.
- Δvenda_30d = 162−152 = +10 (janela/ritmo). No dia 11→12 tinha dado +15; daí a impressão de "17".
- Δestoque = −33 mas só vendeu 9 → confirma que estoque é foto ruidosa, NÃO usar pra inferir venda.

## Fix aplicado (13/jun/2026, working tree do submódulo ecommerce-agent)

- **`agent/api.py`** `/api/havan/history`: calcula `d_vend = Δtotal_periodo`; expõe `vendido_periodo` + `vendido_periodo_delta` por item; ordena a lista de mudanças por `vendido_periodo_delta` (depois venda_30d/pedidos).
- **`dashboard-ui/src/lib/api.ts`**: `HavanChangeItem` ganhou `vendido_periodo?` + `vendido_periodo_delta?`.
- **`dashboard-ui/src/features/havan/index.tsx`**: card e modal de detalhe ganharam coluna **"Vendido (período)"** (headline, em destaque) = venda real entre coletas; a antiga **"Δ Venda"** virou **"Δ 30d (ritmo)"**; nota no subtítulo explicando a diferença.
- Validado: `ruff` ok, `npm run build` ok, serviço reiniciado (systemd), endpoint retornou `vendido_periodo_delta=9` para o case.

**Regra de leitura:** "quanto vendeu desde a última coleta" → **Vendido (período)** / `total_periodo`. "está acelerando?" → venda_30d / venda_7d. Nunca usar Δestoque como venda. Ver [[ecommerce-havan-dashboard]], [[ecommerce-havan-estoque-breakdown]].

## Auditoria 14/jun/2026 — o briefing do Telegram tinha ficado para trás (working tree, NÃO commitado)
CFO viu no briefing matinal "USB-C 3 metros: −7" e no painel +5. Bug: `_run_havan_briefing` (`scheduler.py:~2191`) calculava `venda_novas = Δvenda_30d` (janela móvel → pode dar NEGATIVO) em vez de Δtotal_periodo. USB-C: venda_30d 234→227 (−7) vs total_periodo 1311→1316 (+5). **Fix: trocado para Δtotal_periodo**, alinhado ao card "O que mudou" do painel. Validado contra o banco: nv=+5.
- **2ª divergência corrigida no mesmo passo:** o alerta diário de ruptura (`_run_havan_sync`, `scheduler.py:~2103`) tinha uma heurística ANTIGA de sobre-estoque (cobertura 30d histórica + `venda_30d×2`) divergente do briefing e do painel, que usam o detector canônico `_havan_overstock` (cobertura pela velocidade 7d, decisão CFO jun/2026). **Fix: o alerta de ruptura agora chama `_havan_overstock(prods)`** — fonte única, sem falso-positivo de item acelerando.
- Auditoria varreu api.py/havan_report.py/ai_insights.py/frontend: resto OK (usos de venda_30d/7d são ritmo/cobertura/projeção legítimos; nenhum Δestoque como venda; rótulos do PDF e do card são corretos). "Unidades vendidas (30d)" no PDF é venda real trailing-30d (não delta) → mantido.
