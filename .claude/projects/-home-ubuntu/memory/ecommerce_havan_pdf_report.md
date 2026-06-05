---
name: ecommerce_havan_pdf_report
description: "Relatório executivo PDF p/ o comprador da Havan (agent/havan_report.py) — Plotly+kaleido+PyMuPDF, receita no preço VAREJO Havan, velocidade 7d×30d"
metadata: 
  node_type: memory
  type: project
  originSessionId: a773cfee-b5fc-416a-8e14-90ca69feb266
---

# Relatório PDF executivo para o comprador da Havan (jun/2026, commit `4b0c3ee`)

Documento client-facing p/ enviar ao comprador da Havan e escalar a venda (200+ filiais).
Módulo `agent/havan_report.py` → `build_havan_pdf(snap, horizons, top_filiais, periodo) -> bytes`.

## Stack de geração
- Gráficos: **Plotly → PNG via kaleido** (`fig.to_image(engine="kaleido", scale=2)`). Renderiza headless sob systemd (verificado).
- Tabelas: **`plotly.go.Table` → PNG** (mesma identidade dos gráficos; paginação ~28 linhas).
- Composição: **PyMuPDF/fitz** (capa, faixas de marca navy/amber, cabeçalho/rodapé, nº página).
- **`doc.tobytes(deflate=True, deflate_images=True, garbage=4)` é CRÍTICO** — sem `deflate_images` o PDF fica ~40MB (fitz embute imagem crua); com ele, ~1.3MB.
- `_compress()` (Pillow): cap largura 1150px + paleta 256 cores (gráficos flat encolhem muito).
- **Fonte DejaVu embutida no fitz** (`fontfile=DejaVuSans.ttf`) — os built-in helv/hebo são Latin-1 e viram `?` em em-dash `—` e setas `▲▼`. Nos PNGs Plotly o DejaVu (chromium) já cobre.

## Decisões de negócio (travam as fórmulas)
- **Receita/faturamento = PREÇO DE VENDA HAVAN (varejo)**, não nosso atacado. Doc é p/ o cliente.
  `preco` = varejo Havan (consumidor); `preco_atacado` = o que cobramos da Havan. Helper `_fat_havan(p) = venda_30d × preco`. "Receita pot. 90d" = repor × preco varejo.
- **Velocidade 7d**: a fonte Havan NÃO tem campo rolling-7d → o scraper faz **1 consulta extra** janela `[hoje-7,hoje]` somando `qtdVendasTotais` (`venda_7d` por produto e filial×produto).
- **Ritmo** = `venda_7d × 30/7` vs `venda_30d` (equivalente de 30 dias, NÃO "anualizado"). Limiar ±15% → ▲ acelerando / ▼ desacelerando.
- **Cobertura 30d** = `estoque ÷ venda_30d` (do scraper). A tabela de velocidade mostra TAMBÉM **Cobertura 7d** = `estoque ÷ (venda_7d×30/7)` (`_cobertura_7d`) — cenário conservador, menor que a 30d quando a venda acelera (antecipa ruptura). Ex.: Cabo 12V 2M cob30=1,6m mas cob7=0,5m (+190%).

## Dados
- Nova tabela **`havan_branch_product`** (captured_date, filial, uf, produto, sku, venda_7d, venda_30d, ...): matriz filial×produto p/ ranking por produto. O scraper já tinha `filiais_snapshot` mas só agregados sobreviviam. `set_havan_snapshot` faz `data.pop("filiais_produto")` antes do `json.dumps` (NÃO incha o snapshot). Reader `get_havan_branch_product()`. Populada no havan-sync das 06h.
- `get_havan` refatorado → `_havan_enriched_snapshot(db)` (compartilhado com o relatório p/ usar os mesmos números do dashboard).

## Endpoints / entrega
- **`GET /api/havan/report`** (download) — **SEM sufixo `.pdf`**: o servidor de estáticos do SPA intercepta paths com extensão e devolve index.html. Params: horizons, top_filiais, periodo.
- `POST /api/havan/report/telegram` — envia ao Telegram do CFO (`send_document`, sem caption — vira o filename).
- Scheduler `_loop_havan_report`/`_run_havan_report`: dia 1, 09h30 (após havan-sync 06h), dedup `scheduler_state.havan_report_last_sent` = YYYY-MM, só marca se `send_document` retornar True.
- Frontend: botão "Gerar relatório PDF" + modal (horizontes 30/60/90, top-N filiais) em `features/havan/index.tsx`; baixar (blob) + enviar Telegram (`useHavanReportTelegram`).
- `plotly`+`kaleido`+`pillow` movidos p/ deps core no pyproject.

Relacionado: [[ecommerce_havan_dashboard]], [[ecommerce_havan_backlog_faturamento_transito]], [[ecommerce_ima_sugestao_pedido]].
