---
name: ecommerce-havan-portal-pedidos
description: "ecommerce-agent — portal Havan Fornecedor (cliente.havan.com.br): como logar e extrair Ordens de Compra abertas (valor + semana de entrega + prazo pgto) para o fluxo de caixa"
metadata: 
  node_type: memory
  type: reference
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Portal Havan Fornecedor — pedidos de compra (mapeado 13/jun/2026)

Portal de PEDIDOS (≠ do portal de relatórios `fornecedor.apps.havan.com.br` que o
`agent/havan_scraper.py` já usa). Base: **`https://cliente.havan.com.br/Fornecedor`**.
Mesmas credenciais `HAVAN_USER` (email) / `HAVAN_PASS` do .env.

## Login (contrato exato)
POST `/Fornecedor/Login/FazerLogin?Length=5`, form-urlencoded:
`TipoLogin=1` (1=Email, 0=CNPJ), `Documento=`, `Email=<HAVAN_USER>`, `SenhaMd5=<HAVAN_PASS>`
(⚠️ campo chama SenhaMd5 mas vai em **TEXTO PURO**, não MD5), `X-Requested-With=XMLHttpRequest`.
Sucesso → sessão no cookie do context; redireciona p/ `/Fornecedor/informativo/`. Usar Playwright
`ctx.request.post(...)` e depois `ctx.request`/`pg.goto` com o mesmo context. Páginas têm conexão
persistente → usar `wait_until="domcontentloaded"` (networkidle trava).

## Menu (links úteis)
`/Fornecedor/PedidoCompra/Index` (pedidos), `/Fornecedor/ExtratoFinanceiro/Index` (financeiro —
recebíveis reais, ainda não explorado), `/Fornecedor/Download`, `/Fornecedor/DadosCadastrais`.

## Lista de Ordens de Compra
POST **`/Fornecedor/PedidoCompra/GridIndexPedidoCompra`** (form serializado de `#frm-indexpedidocompra`):
- Campos: `OpcaoCompradorPedidoCompra`, `Pedido`, `OpcaoSituacaoPedidoCompra` (T=Todos/Fechados/Liberados),
  `NotaFiscal`, `OpcaoStatusNotaFiscal` (0=Todos; "Sem nota"=não faturado), `OpcaoAnoEntregaPedidoCompra{Inicio,Final}`
  (ano), `OpcaoSemanaAnoEntregaPedidoCompra{Inicio,Final}` (nº da semana 1-52), `OpcaoDataEmissao...`.
- **Exige ao menos 1 filtro de período** e o período **máx = 1 ano** (ex.: 2026 sem1→sem52 OK; →2027 falha).
- `+ &gerarExcel=true` → devolve **xlsx** `PedidosCompra.xlsx` (colunas: Pedido, Status do pedido
  [Liberado/Fechado], Versão, **Semana de entrega** "2026 - 27 - 05/07/2026 à 11/07/2026", Nota fiscal,
  **Status da nota fiscal** [Sem nota/Concluída], Agendamento, Comprador). **Sem valor/qtd no xlsx.**
- Sem gerarExcel → HTML em `div`s. Cada bloco tem o **idPedido** e links de arquivos via
  `/Fornecedor/PedidoCompra/SetaPedidoVisualizadoBaixado?idPedido=..&arquivo=Ordem%20de%20compra&extensaoArquivo=PDF&idArquivo=..&idRelacao=..`
  (⚠️ o nome "SetaPedidoVisualizadoBaixado" marca o pedido como visto/baixado — efeito colateral esperado).

## PDF "Ordem de compra" (tem o VALOR + prazo)
GET o link SetaPedido... (auth) → PDF (PyMuPDF/fitz parseia limpo, ~2 pág). Contém:
- **Itens**: Item, Código, Descrição, Referência, **Quantidade**, UN, **Pr.Unitário**, **Preço Total**.
- **Valor total** da OC (ex.: 2026-62310 = R$ 69.875 = CABO USB-C 3M ×250 + SUPORTE SLIM PRO 2 ×675).
- **Semana de entrega** "27 (05/07/26 - 11/07/26)".
- **Prazo de pagamento: 121 dia(s)** (explícito por PO, a partir do RECEBIMENTO da mercadoria no CD;
  vide INSTRUÇÃO ADICIONAL N.1). CFO disse 120; usar o valor da própria PO.
- Fornecedor = CONNECT IMPORTAÇÃO (XConnect matriz RS 57.710.730/0001-01); destinatário HAVAN CDH BARRA VELHA.

## No fluxo de caixa — IMPLEMENTADO 13/jun (commit `148ba73`)
Open PO (Status NF="Sem nota") → **recebimento = fim da semana de entrega + prazo_pgto (121d do PDF)**;
**valor = valor total da OC** (do PDF). Linha **"Havan a faturar"** no cashflow.
- **`agent/havan_pedidos_scraper.py`** `fetch_pedidos_abertos(cache, years)`: login + GridIndexPedidoCompra
  (HTML, ano corrente+próximo) + parse blocos (regex `_BLOCK_RE`, tolera tags e `&amp;`) + baixa/parseia
  PDF (cache por id_arquivo). Detecta login falho (raise → não sobrescreve). **Precisa do `pg.goto`
  PedidoCompra/Index ANTES do POST do grid** (senão grid vem vazio).
- Tabela singleton `havan_pedidos_snapshot` + `get/set_havan_pedidos_snapshot` + `get_havan_pedidos_cache`.
- Scheduler: `_run_havan_pedidos_sync` ao fim da coleta Havan diária (try/except).
- `get_cashflow`: `receita_havan_aberto_est` por mês (de `_havan_aberto_schedule`); **reconciliação**:
  o modelo de demanda `_havan_revenue_schedule` passa a começar em `max(cobertura, dias_até_última
  entrega aberta)` — só após as entregas dos abertos, p/ não dobrar. Campo `havan_aberto_total`.
  Endpoints `GET/POST /api/havan/pedidos-abertos(/sync)`.
- Validado: 8 OCs = **R$ 832.080** (receb. Out 183k/Nov 288k/Dez 361k); demanda caiu p/ R$ 15k.
Ver [[ecommerce-dre-competencia-revisar]] (fila CFO item 3 ✅) e [[ecommerce-havan-backlog-faturamento-transito]].

## ✅ FEITO 13/jun (commit `bc76c80`) — quantidades das OCs abertas na simulação Havan
`fetch_pedidos_abertos` agora guarda os **itens** de cada OC (`_parse_oc_pdf` já os extraía) e
devolve **`itens_agregados`** (qtd por produto somando as OCs). Cache do scraper inclui itens
(`get_havan_pedidos_cache`). `/api/havan/simulacao-pedido`: cada linha ganha **`qtd_ocs`**
(match por PREFIXO — a descrição da OC é prefixo truncado ~24 chars do nome do snapshot;
`_pn.startswith(d)`) + `oc_total_qtd`. Frontend /simulacao-havan: botão "🏬 OCs em aberto (N)"
preenche todas as qtds com as das OCs; cada linha tem "OC: X" clicável p/ usar só aquela.
Validado: 8 OCs → 10.775 un (CABO 12V 3M 2930, SLIM PRO 2 1350, CASE 500, USB-C 3M 1595, ...).
⚠️ Se a Havan mudar o padrão de nomes, o match por prefixo pode falhar (revisar). USBC 5M e o
Cabo USB 2M novo ficam com qtd_ocs=0 (não pedidos).

## ✅ FEITO 13/jun (commit `5713349`) — saída de IMPOSTO do "Havan a faturar"
A entrada "Havan a faturar" é BRUTA; agora há a SAÍDA de imposto correspondente (senão o caixa
superestima). Helper **`_havan_aberto_impostos_por_mes(conn, pedidos, snap_produtos)`**: por OC,
**ICMS líquido (débito 12% − crédito de insumos por produto via `_havan_unit_cost`) + PIS/COFINS
3,65% + IRPJ/CSLL 2,28%** sobre o atacado, no mês de PAGAMENTO (~mês seguinte à entrega). Crédito por
item: casa a descrição TRUNCADA da OC com o nome completo do snapshot (prefixo) → `_HAVAN_CMV_MAP` →
product_costs. `get_cashflow`: `_havan_aberto_imposto_schedule` subtraído do saldo; campos
`havan_aberto_imposto_est`/`havan_aberto_imposto_total`; **linha SEPARADA** "Imposto Havan a faturar"
no detalhe do mês + barra própria no gráfico. Provisão: saída no day-by-day (~dia 20, os que caem no
horizonte) + `imposto_total` no banner. Validado: R$ 832k → imposto **R$ 113.210 (13,6%)**, pago
Jul-Set ANTES do recebimento Out-Dez.

## ⏳ PENDENTE (CFO 13/jun) — revisar se há CRÉDITO DE ICMS nas VENTOSAS
Na implementação do imposto/CMV das OCs (helper `_havan_oc_produto_econ`) assumi que as ventosas
novas (R$ 3,05, à vista) são de fornecedor **Simples Nacional → SEM crédito de ICMS** (excluí o
componente 122 do crédito, igual aos cabos). **Confirmar com o CFO/contador** se realmente não há
crédito — se houver, incluir o crédito da ventosa no `econ["credito"]` (reduz o ICMS líquido / o
imposto do "Havan a faturar"). Hoje está conservador (sem crédito). Idem rever o crédito do imã (hoje
INCLUÍDO no crédito da venda — correto, é input real) e dos corpos.

## ⏳ PENDENTE (CFO 13/jun) — INSUMOS A COMPRAR para entregar as OCs abertas (CMV no fluxo)
Falta a **saída de caixa dos insumos que precisamos COMPRAR** para produzir/entregar os pedidos em
aberto. **NÃO considerar o que já temos em estoque (ex.: imãs)** — só o que falta adquirir. Pensar e
planejar, por produto da OC, o que precisa ser comprado (corpos injetados, cabos, suportes, ventosas,
etc.), o custo e **quando sai do caixa** (lead do fornecedor, antes da entrega → antes do recebimento
+121d). É o CMV/compra correspondente às OCs abertas. Cuidado p/ não dobrar com o CMV que já entra no
fluxo (cmv_base/cmv_imas) nem com o `a_pagar_op`. A simulação já tem CMV por produto
(`_havan_unit_cost`/`get_havan_simulacao_pedido`) → reaproveitar para estimar o custo; o desafio é o
TIMING (quando compramos) e abater o que já está em estoque. A conceber. Ver [[ecommerce-import-cost]],
[[ecommerce-ima-sugestao-pedido]] (estoque/runway de imãs).

## ⏳ PENDENTE (CFO 14/jun) — DETALHE do card "Havan a faturar" no Fluxo de Caixa
No Fluxo de Caixa, o card que mostra **quanto há de Havan a faturar (pedidos/OCs em aberto)** deve ganhar
o **DETALHE (drill-down) de (a) IMPOSTOS e (b) INSUMOS que saem do caixa ANTES de receber os pedidos** —
ou seja, o **investimento/fluxo de saída que a empresa faz para conseguir ENTREGAR as OCs abertas**.
Enquadrar como "para faturar R$ X de OCs abertas, sai R$ Y de imposto (Jul-Set) + R$ Z de insumos a
comprar (antes da entrega), e só depois entra o recebimento (entrega+121d)". Já existe o backend dos
impostos (`_havan_aberto_impostos_por_mes`, linha separada "Imposto Havan a faturar") — falta o
**drill-down clicável** com a composição (por OC/produto: ICMS líq, PIS/COFINS, IRPJ/CSLL) e o
**INSUMOS a comprar** (item pendente acima — CMV/compra das OCs, abatendo estoque, com timing). Padrão de
drill-down: DetailDialog já existe no projeto. Ver [[ecommerce-cashflow-financiado]].
