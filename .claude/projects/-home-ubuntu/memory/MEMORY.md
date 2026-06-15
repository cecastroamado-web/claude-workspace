# Memória dos Projetos

## Projetos Ativos

### 0. finance-agent (`/home/ubuntu/finance-agent`) — 5 fases completas (2026-05-18)
- Dashboard finanças pessoais (PF) com Open Finance Brasil (Pluggy) + Claude AI + Telegram + React/FastAPI
- 57 arquivos, 16 módulos Python, ~30 endpoints REST, 13 loops async, 13 tabelas SQLite
- Custo real: ~R$ 6-16/mês (Pluggy + Claude com prompt caching 90%)
- Ver [finance-agent project](finance_agent_project.md) — PLAN.md e README.md no projeto têm detalhes técnicos

### 1. email-agent (`/home/ubuntu/email-agent`)
- Assistente de email inteligente: Gmail + **Telegram** + Claude AI
- Python 3.12, linguagem natural via Telegram Bot API
- **Módulo financeiro** (`agent/financeiro/`) — lê abas Banco de 10 contas em 6 países
- Comandos via Telegram: linguagem natural (pagamentos, financeiro, status, etc.)
- **Migrado de WhatsApp para Telegram** (mar/2026)
- [Estrutura aba Receitas](email_agent_receitas_structure.md) — colunas, separação serviço vs mídia, fluxo media_monthly → projeção

#### AI Features (mar/2026)
- **Prompt Caching**: system prompt como list[dict] com cache_control ephemeral (tools.py, reports.py)
- **Streaming**: messages.stream() com editMessageText progressivo (message_handler.py, telegram_client.py)
- **History Summarization**: >12 msgs → Haiku resume, mantém últimas 6 (message_handler.py)
- **Extended Thinking**: thinking budget 2048 para relatórios complexos (burn, fluxo, anomalias, projecao, pais)
- **Proactive Insights**: 3 insights diários pós-relatório financeiro (scheduler.py → reports.py)
- **Mem0**: memória persistente local com Qdrant + HuggingFace embeddings (agent/memory.py)

#### Internacional — auditoria completa (mai/2026)
- Ver [Critérios Financeiros Internacional](email_fluxo_diario_intl.md) — todas as 20 páginas auditadas, IVA pass-through, Margem Op/Líq sem dívida, câmbio histórico em todos os contextos com data conhecida, BJ/payment_terms/country_pay_day na cadeia de previsão.
- Doc completo no projeto: `docs/criterios_financeiros_internacional.md` (referência ao adicionar páginas).

#### Validação Brasil — em andamento (mai/2026)
- Ver [Bugs do dashboard mai/2026](email_dashboard_bugs_mai2026.md) — A1 (plataformas mídia: FACEBOOKADS classificado como Google) e A2 (vendedor_2 com placeholder "0" e duplicado) corrigidos. Pendência aberta: KPI Margem Bruta com numerador/denominador inconsistente.
- Ver [DRE Brasil — categorize_dre](email_dre_brasil_categorization.md) — função `categorize_dre()` em `despesas_brasil_reader.py` com aliases (RECURSOS HUMANOS→RH) e fallback por pacote_nome (CAPEX→INVESTIMENTO etc). Linha "Outros" caiu de R$ 7,78M → R$ 10k acumulado 2024→2026. Validação 10/mai: total R$ 23,85M confere; 2 sub_centros pendentes ("COMERCIAL INT" + "PASSAGEM AEREA") cosméticos (R$ 5,8k).
- Ver [Mídia Brasil — migração para Reweb Corp](email_diagnostico_midia_2026.md) — queda Mídia Brasil 2026 é MIGRAÇÃO de canal (cartão Intl → wire→Reweb), NÃO saída de clientes. Consolidado Brasil+wire: 7,83M (24)→6,48M (25)→5,86M (26 anualiz., -25%). DRE Brasil subestima CMV em 2026.
- Ver [Grupo "Sem Dados" do BQ](email_grupo_sem_dados.md) — BigQuery devolve string literal "Sem Dados" no campo Grupo, mascarava 47,8% da receita Brasil. Helper `_grupo_is_empty()` no app.py trata como vazio.
- Brasil — Fluxo Caixa (`app.py:5984`): receitas carregavam só desde 2024 enquanto despesas vinham desde 2010 → 2022-2023 apareciam com R$ 0 de entrada e R$ 1,7M/mês de saída. Fix: ambos desde 2022. Total acumulado real: déficit -R$ 6,6M (não -R$ 37,3M).
- Brasil — Dossiê (`brasil_reports.report_dossie_cliente`): ticket médio era LTV ÷ todos os meses, deflacionava quando havia mês com fee=0. Fix: divide só por meses com fee > 0. Display também trata "Sem Dados" como "—" via `_grupo_is_empty()`.
- CAC por País (`bq_billing_reader.get_brasil_novos_clientes`): SQL aliasou `p.prim_comp AS prim_competencia` mas Python lia `r.get("prim_comp")` — coluna 1ª Competência ficava vazia. Fix: ler `prim_competencia`. CAC Brasil últimos 6m oscilou R$ 3,8k–19,3k/cliente (média ~R$ 10k).
- Global — Ranking (`bq_billing_reader._intl_billing_query`): `DATE_TRUNC(data, MONTH)` falhava silenciosamente porque `data` é STRING no BQ → 0 transações internacionais. Página mostrava só Brasil desde sempre. Fix: `DATE_TRUNC(DATE(data), MONTH)`. Hoje retorna BR + 6 LATAM corretamente.
- Cache vazio do Sheets API (`disk_cache` em `sheets_utils.py`): quando a API devolvia `values=[]` sem HTTP error, o cache persistia 0 linhas por 10min e todo dashboard mostrava zeros. Fix: `disk_cache` aceita `cache_predicate` opcional; `_read_sheet_rows` re-tenta após 2s; `get_despesas_brasil`/`get_receitas_brasil`/`get_saldo_brasil` passam `cache_predicate=lambda r: bool(r)`.
- Doc no projeto: `docs/validacao_dashboard_maio2026.md` — A1, A2, M2, M3, M4 ✅ em 05/mai. DRE + Ranking + Fluxo Caixa + Dossiê + CAC + Provisão + Global Ranking + cache vazio corrigidos em 07/mai. Críticos C1-C4 ainda pendentes.
- Ver [Brasil competência vs caixa](email_fluxo_brasil_competencia_vs_caixa.md) — modo Avançado do Fluxo Diário Brasil mudou default histórico→PROG em mai/2026 (média histórica subestimava em ~R$ 250k/mês). Pendente: filtrar PROG que vai ser parcelado/renegociado (financiamento via imposto) para não superestimar.
- Ver [WIRE Intercompany — auto-detecção](email_wire_intercompany.md) — wire Motorleads↔Motorleads (ex.: Reweb Peru→Colombia) detectada por benef+desc, marcada `intercompany=True`, excluída do fluxo líquido por país em `app.py:1624`. EOM consolidado nunca foi afetado.
- Ver ["Não Cobrar" em Data Prometida](email_data_prometida_nao_cobrar.md) — convenção operacional: cliente não aceita cobrança do time, CFO cobra pessoalmente. NÃO é exclusão — fatura continua válida no fluxo/atrasados. Ex.: Renault Mexico (faturas 9928/9929).
- Ver [Filtros e overrides da Projeção mensal](email_projection_filtros.md) — limite PDD subiu 60→120d (mai/2026); override Irrecuperável via recv_state.json marca inv.juridico=True; radio include_overdue no topo da página Projeção (skip/today/eom).
- Ver [Projeção EOM subindo após pagamento = baixa incompleta](email_projecao_baixa_incompleta.md) — WIRE In no banco sem Data Pagamento na aba Receitas infla EOM (paid_periods não filtra). **+ bug inverso (corrigido 08/jun/2026, `895149c`):** paid_periods por (cliente,ano,mês) escondia da Projeção fatura em aberto quando uma irmã do mesmo mês era paga; agora exclui por nº de fatura (col M da Receitas). Caso Alden/Kia 9941 vs 9942; surfaceou 3 recebíveis reais (5→8).
- Ver [Dashboard ops (Streamlit 8502)](email_dashboard_ops.md) — launch manual porta 8502, "Forçar Atualização" só limpa cache (mudança de código exige RESTART do processo), rate limit Sheets 60/min, col M=Fatura na Receitas.

### 2. ecommerce-agent (`/home/ubuntu/ecommerce-agent`)
- Agente e-commerce para venda de acessórios Starlink
- Duas empresas: XConnect (Lucro Presumido, ML) + Aviation (Simples Nacional, ML+Shopify)
- Integrações: Bling API v3 (NF-e), Banco Inter (boletos mTLS), Google Sheets, Telegram
- Pipeline: parse(Claude AI) → Bling(contato+pedido+NF-e) → Inter(boleto) → Sheets(A RECEBER) → Telegram(PDFs)
- Porta **8504** = webhook Telegram (`WEBHOOK_PORT` via `agent/config.py`); porta **8510** = dashboard FastAPI + UI React (`dashboard-ui/`)
- Mesmo processo `python3 main.py` serve ambas as portas
- **Gerenciado pelo systemd**: `ecommerce-agent.service` com `Restart=always` — NÃO usar nohup (causa conflito de porta e loop de restart)
- **Reiniciar**: `kill <PID>` — o systemd reinicia automaticamente em até 30s
- **Encontrar PID**: `ss -tlnp | grep 8510 | grep -oP 'pid=\K[0-9]+'`
- **Rebuild frontend**: `cd /home/ubuntu/ecommerce-agent/dashboard-ui && npm run build`
- **Logs do serviço**: `journalctl -u ecommerce-agent.service -n 50 --no-pager`
- **Status**: `systemctl status ecommerce-agent.service`

#### Bugs corrigidos
- [sale_fee ML é por unidade, multiplicar por qty](ecommerce_ml_sale_fee_qty.md) — bug histórico afetando 499 pedidos qty>1 (mai/2026)
- [customer-ranking refatorado p/ usar compute_profitability](ecommerce_customer_ranking_refactor.md) — bug B2B inflando lucro Havan em R$ 42k (mai/2026)
- [NFs Havan 547/552 — cancelamento/reemissão por regime (NÃO devolução)](ecommerce_havan_devolucoes.md) — canceladas 2025 (Simples) → reemitidas 2026 (LP), produto entregue jan/2026; Havan não compra de Simples. NF sem entrada no sheet pode ser devolução OU reemissão — verificar. Recebido jan/2026 é entrega real (sinal de sobre-estoque é válido).
- [Dashboard do canal Havan](ecommerce_havan_dashboard.md) — scraper 1×/dia 06h, margem/P&L (c/ comissão Mark 1% s/ bruto), conciliação+em-trânsito, analytics de expansão (rede 196) + potencial no tempo + página /simulacao-havan (mix 22k editável, cenários imã/ventosa/cabos, impacto no lucro, PDF). Pendências em [[ecommerce-havan-pendencias]].
- [Quebra do estoque Havan](ecommerce_havan_estoque_breakdown.md) — CONCLUÍDO jun/2026: Total Havan (prateleira+CD+trânsito=previsto) é a base OFICIAL de estoque/cobertura/ruptura em todos os cards; backfill impossível (estoque é foto de agora); em-trânsito na reposição + ordenação por filial-need; KPI "Ruptura na prateleira"; indicador ⚠ fonte inconsistente.
- [Havan — Vendido (período) vs janela 30d](ecommerce_havan_vendido_periodo.md) — card "O que mudou": venda_30d é janela móvel (qtdVenda30Dias), total_periodo é venda real; nova coluna Vendido (período) (jun/2026).
- [Relatório PDF p/ comprador da Havan](ecommerce_havan_pdf_report.md) — `agent/havan_report.py` (Plotly+kaleido+PyMuPDF), `GET /api/havan/report` (sem `.pdf`!) + Telegram + mensal. **Receita = preço VAREJO Havan** (não atacado). Velocidade 7d (janela extra no scraper) + ritmo 7d×30/7. Tabela `havan_branch_product` (filial×produto). `deflate_images` essencial (40MB→1.3MB).
- [Override de desconto contratual por NF](ecommerce_nfe_discount_override.md) — tabela nfe_discount_override + _compute_customer_discount; usado quando retenção incide sobre valor original pré-devolução (ex.: HAVAN NF 561, R$ 5.488 sobre R$ 274.400 → líquido R$ 118.662). Bugs corrigidos: vendor-roi sem customer_name; commission-report sem campo desconto_cliente.
- [Aviation 11% fixo na apuração de comissões](ecommerce_comissoes_aviation_rate.md) — decisão de negócio: comissão usa rate estável, não a real do simples_rate_history
- [Regras de consulta Bling API v3](ecommerce_bling_rules.md) — corte 16/mar/2026 XConnect, linkDanfe, exclusões EBAZAR, OAuth setup
- **CFOP filter no commission-report** (mai/2026): incluído `_NFE_CFOP_VENDA_SQL` igual ao vendor-roi e profitability — exclui devoluções/remessas (CFOPs 4xxx, 59xx, 69xx). Validação: contagem de NFs do Rafael agora bate com vendor-roi (23=23)
- [ICMS XConnect — matriz RS / filial SP + alíquotas reais](ecommerce_icms_aliquotas_reais.md) — Matriz XConnect em RS (CNPJ 57.710.730/0001-01) emite as NFs ML, filial em **SP (CNPJ 57.710.730/0004-46, 737 NFs)**. SEMPRE usar `valor_icms` da NF. RS: intra=17%, S/SE=12%, demais=7%; SP: intra=18%, S/SE=12%, demais=7%. **Fix jun/2026**: bug do cUF (api.py gravava código IBGE "43"/"35", metrics.py comparava "RS"/"SP" → branch RS estava morto). Corrigido `_IBGE_UF`/`_uf_from_invoice_key` (api.py) + branch SP (metrics.py). Impacto real só XConnect LP 2026 ICMS=0 (18 RS + 9 SP). Limitação: DIFAL=0 quando sem ml_nfe.

#### Regras de Receita — Aviation (validado mar/2026)
- **ML direto** → tabela `ml_orders_sync`, status=`paid` apenas (excluir `cancelled` mesmo que pagos — são reembolsados)
- **Bling NF-e** → tabela `bling_nfe`, `situacao=6` (emitidas direto no Bling: manuais + Shopify), excluir EBAZAR (transferências de estoque ML)
- **NFS-e** → tabela `bling_nfse`, `situacao IN (1,3)` (1=emitida, 3=autorizada município). XConnect tem 403 (sem NFS-e). Aviation tem sync funcionando.
- `bling_orders` (pedidos/vendas) → NÃO usar como receita, não é confiável
- `situacao=2` Bling = autorizada SEFAZ, pode duplicar ML → não usar
- ML dashboard mostra gross (inclui cancelados-pagos); nossa contagem é net (só paid não cancelados) — diferença é normal
- Transferências EBAZAR = NF-e cujo destinatário contém "EBAZAR" → excluir sempre

#### AI Features (mar/2026)
- **Prompt Caching**: system prompt + tools com cache_control ephemeral
- **Smart Model Routing**: queries simples → Haiku, complexas → Sonnet (CLAUDE_MODEL_FAST)
- **Streaming**: messages.stream() com editMessageText progressivo
- **History Summarization**: >12 msgs → Haiku resume, mantém últimas 6
- **Extended Thinking**: thinking budget 1024 para parsing de pedidos complexos (>200 chars ou >3 linhas)
- **Instructor**: structured output com Pydantic models para parsing de pedidos (parser.py)
- **Mem0**: memória persistente local (agent/memory.py)

#### Alertas CFO + scheduler
- Ver [Alertas CFO ecommerce-agent](ecommerce_alertas_cfo.md) — inventário dos jobs (promo ML, vendas diárias, saldo, anomalias, NF-e, comissão, catálogo, ima-depletion), thresholds, state efêmero, catch-up no restart
- Ver [CI GitHub do ecommerce-agent](ecommerce_ci_github.md) — ruff+mypy+pytest no push; ALTER manual no DB exige migração no db.py; side effects de import guardados por DB_PATH.exists(); fixtures forçam credenciais (poluição de coleta)
- Ver [Feedback: testar só o que mudou](feedback_testar_so_o_que_mudou.md) — verificação 1:1 com o diff; pipeline de pedido Telegram NÃO está finalizado/validado (jun/2026), não usar como smoke test

#### Fluxo de Caixa — simulações de financiamento (jun/2026)
- Ver [Fluxo de caixa financiado](ecommerce_cashflow_financiado.md) — defaults: parcelamento impostos 12 meses/Selic 1,15%/início mês 01; spread Sicredi 0,40%. Empréstimo Capital de Giro **refatorado jun/2026 p/ modo único** (PV+parcelas+garantia+Selic+spread) com seletor **Price/SAC** (`emprestimo_giro_sistema`); removidos modos banco-propõe/comparar. Parcelamento de impostos agora tem **breakdown por tipo** por adesão (expansível). Proposta real Sicredi: garantia 600k → giro 900k em 60x = 1,53% a.m. **Card "Aplicado em Renda Fixa" na Visão Geral** (`/api/overview.saldo_rf_atual`; XConnect R$ 300k).
- Ver [Visão Geral — categorias de caixa](ecommerce_overview_categorias.md) — identidade **Variação de Caixa = Δ saldo do banco** e os 4 baldes. Categoria **IMPOSTO A RECUPERAR** (`CATEGORIAS_RECUPERAVEL`): valor que saiu mas volta (imposto de parcelamento negado, reembolso pedido) — fora do P&L (financiamento), entra no fluxo de Investimentos via `recuperaveis_liquido_periodo`. Transferências entre contas próprias devem zerar; se não zeram, é perna faltando/categoria errada.

#### ML NF-e / DIFAL — implementado mar/2026
- Tabela `ml_nfe`: NF-e emitidas pelo faturador ML (Fulfillment orders). `invoice_status='not_found'` = entrega direta (drop-off/self_service), NF-e emitida pelo vendedor via Bling
- ML só emite NF-e para pedidos `logistic_type=fulfillment` (MELI warehouse)
- Drop-off/self_service: 87 XConnect + 5 Aviation com `invoice_status='not_found'` → NF-e no `bling_nfe`
- XConnect 2025: Simples Nacional → icms/difal = 0. Tornou-se LP em 2026, DIFAL aparece a partir de 2026
- Endpoints: `/api/ml-nfe`, `/api/ml-nfe/sync`, `/api/ml-nfe/sync-status`, `/api/ml-nfe/difal`, `/api/ml-nfe/difal/export`
- `sync-status` retorna: `synced` (NF-e ML), `not_found` (entrega direta), `pending` (em trânsito), `total`
- DIFAL via GNRE coletado pelo ML em nome do vendedor: `valor_icms_ufdest` (destino) + `valor_fcp_ufdest` (FCP)
- XLSX export para contador: 3 abas — Resumo Mensal, Por Estado, Detalhe Mês×Estado

#### Provisão de Caixa — modelo antecipação (abr/2026)
- Ver [ecommerce_provisao_model.md](ecommerce_provisao_model.md) — lógica completa do `/api/provisao`
- Ver [ecommerce_icms_model.md](ecommerce_icms_model.md) — _calc_icms_net_month, DIFAL, modelo por empresa, bugs corrigidos abr-mai/2026 — **BUG CRÍTICO MAI/2026**: query de crédito sem component_id somava versões históricas → crédito inflado. Fix: subconsulta correlacionada com component_id. Resultado validado abril: débito R$29.416, crédito R$13.377, líquido R$16.039.
- Ver [ecommerce_ads_fix.md](ecommerce_ads_fix.md) — ADS no cashflow: extrapolar só com ≥10 dias; senão usar mês anterior (fix mai/2026)
- Ver [ecommerce_sheets_append_fix.md](ecommerce_sheets_append_fix.md) — append_sale_row usa insertDimension+update, NÃO values().append() (fix mai/2026)
- Ver [Custo de importação — calculadora + correção ICMS](ecommerce_import_cost.md) — `agent/import_cost.py` (módulo+CLI+xlsx), endpoints `/api/import-cost(/agx)`, lógica TTD-SC (ICMS destacado 4% pass-through vs recolhido 1,2%), IPI s/ VA+II. Nacionalização do pedido de imãs corrigida (~182k→168k); banner azul "pedido realizado" removido, card amber com simulação (jun/2026)
- Ver [Sugestão de pedido de imãs — demanda ML+Havan](ecommerce_ima_sugestao_pedido.md) — velocidade agora combina ML (`ml_orders_sync`) + Havan sell-out do case (snapshot, MA 3m, exclui mês corrente); runway combinado; sugestão de qtd p/ cobertura lead+6m (decisões CFO). Corrige texto enganoso do alerta + erro de estimar Havan por valor÷R$295 (jun/2026)
- Ver [Faturamento Havan em trânsito](ecommerce_havan_backlog_faturamento_transito.md) — em-trânsito (`6127d17`/`7ad4de9`, caveat snapshot `recebido` SUBCONTA) + **card detalhe "Havan a faturar"** (`/api/havan/aberto-detalhe`): desencaixe mês a mês com lançamentos (insumos+impostos por OC, expansível), antecipação de OCs (modal + painel de fluxo, param `antecipa_havan_aberto[_ocs]`), PDFs. Prazos insumos: cabos à vista, ventosas em bloco único 19/06, bases Gedeval 120d. (15/jun/2026)
- Ver [Portal Havan Fornecedor — pedidos de compra](ecommerce_havan_portal_pedidos.md) — `cliente.havan.com.br/Fornecedor` (≠ portal de relatórios). Login decifrado (TipoLogin=1, SenhaMd5=texto puro). POST `GridIndexPedidoCompra` (+gerarExcel=true → xlsx) lista OCs com **semana de entrega** + status NF; PDF "Ordem de compra" tem **valor + qtd + prazo pgto 121d**. Base p/ projetar pedidos em aberto no fluxo (recebimento = entrega + 121d). A implementar.
- Ver [Sync de itens NF-e — bug destrutivo corrigido](ecommerce_nfe_items_sync_fix.md) — item_id='' colapsava itens + job deletava NFs com gap legítimo (lucro Havan NF 561 inflado 34,5k→real 7,5k). Lock `nfe_items_lock` (561 travada — devolução), alerta Telegram de gap, replace_nfe_items preserva overrides (05/jun/2026)
- Ver [Em-trânsito via estoque — decisão pendente](ecommerce_havan_transito_estoque.md) — relatório de recebimento da Havan ATRASA vs estoque (jun/2026: estoque subiu, recebido_mensais junho=0). Decisão: trocar timer 15d por detecção via aumento de estoque. **PENDENTE 12/jun**: esperar sync 06h; se recebido junho continuar 0, implementar (modelo pronto). Nada codado ainda.
- Conciliação Havan: **"não-entrada"/em-trânsito (R$) agora respeita a confirmação manual de entrega** (`entregue_ate`) — vinha de `nosso−recebido_oficial` (relatório oficial atrasa) e nunca baixava ao marcar entrega; agora vem do valor realmente em trânsito (`_havan_em_transito_valor_por_mes`). Fix do botão "Marcar entregue" (stopPropagation — clique abria o modal do card). Commit `4d4c3d5` (11/jun/2026).
- Ver [Briefing matinal CFO — PLANO](ecommerce_briefing_matinal_cfo.md) — proposta (não implementada) de um "Bom dia, CFO" único no Telegram ~06h30 pós-coleta: Caixa hoje, Recebíveis/Havan, Vendas (inclui **vendas por item D-1**), Ruptura, Impostos, Saúde de dados. ✅já existe vs 🆕novo; decisões do CFO pendentes (escopo MVP/horário/limiar).
- Ver [Mês vigente do fluxo não desconta realizado](ecommerce_cashflow_mes_vigente_realizado.md) — **PENDENTE 12/jun**: projeção do mês corrente mistura saldo_real (já reflete dividendos/opex pagos) com estimativas de mês cheio → dividendo modelado sem reconciliar com já-pago + CMV/impostos/ADS podem subtrair 2×. Estudar tb provisão+gráfico.
- Ver [Revisar DRE + competência + AUDITORIA resultado operacional](ecommerce_dre_competencia_revisar.md) — **PENDENTE 12/jun**: CFO suspeita que o resultado operacional conta CUSTOS EM DOBRO (CMV/insumo lançado na compra E na venda). Auditoria completa pedida. + crédito ICMS por competência de venda (rentabilidade) vs entrada (fiscal). Confere: ICMS RS matriz maio R$ 27.552 (débito 37.378 dominado pela Havan − crédito fiscal 9.826).
- Ver [Receita sem documento fiscal — auditoria](ecommerce_receita_sem_nota.md) — vendas recebidas no caixa SEM NF-e (favorecido≠X Connect Import no Sheets). Sem-nota CONFIRMADO ≥R$5k = R$615.463 (Israel, Startecno, Star2go-excedente, etc.); 21 a confirmar (~R$252k) + cauda <R$5k. Match por NOME (não valor) vs NF-e situacao=6 E NFS-e. CMV por ANO (2024~31%/2025~43%/2026~38%), sem comissão ML/imposto/overhead. XLSX `receita_sem_nota_REVISAR.xlsx` (4 abas) no Telegram p/ CFO marcar S/N. **Nada gravado no banco até CFO fechar.**
- Ver [Líquido Havan por SKU pós-carry 121d](ecommerce_havan_liquido_carry_sku.md) — análise PLANEJADA (não codada): líquido = preço atacado − CMV(China 2026) − comissão Mark 1% − impostos LP − carry 121d (~Selic×121/365 ≈ 5pp). CFO avalia líquido Havan 20-32% (ok p/ atacadista grande). B2B avulso tem + margem (ativação embutida). Objetivo: piso de margem por SKU p/ renegociar. Aguarda XLSX sem-nota.

#### Dashboard Cashflow — implementado mar/2026 (`agent/api.py`, `dashboard-ui/src/features/cashflow/`)
- **Pipeline cases 66mm**: produzíveis (imãs estoque ÷4) + em trânsito + ML (auto-fetch no load)
- **ML stock auto-fetch**: `asyncio.wait_for(..., timeout=25)` no início de `/api/cashflow` — sem precisar clicar ↻ ML
- **Em trânsito**: lido da aba "Saídas 2026" do Excel de imãs (Drive ID `1a0G6lDQjMc_B4sIOcVT3iKwZtXKl_mTU`), coluna `Destino = "em transito"`
- **IM-88 varejo**: `_IM88_MONTHLY_CASES = 0` (negociação não fechada — sem projeção no fluxo). Reativar quando negociar.
- **Pedido imãs**: 40k imãs = 10k cases, USD 68.800 × R$5,25 = R$361.200; nacionalização R$148.200 (~60d do embarque); pagamento imãs R$361.200 (270d do embarque). **EDITÁVEL/PERSISTIDO desde jun/2026** (commit 820b450): card "Pedido de Imãs Importados" no Fluxo de Caixa com toggle "Considerar no fluxo" + Data do pedido + Data do embarque. Tabela singleton `ima_order_config` (enabled/order_date/embarque_date) + `GET/POST /api/ima-order-config`; `/api/cashflow` lê via `_get_ima_order_cfg()`. Embarque recalcula chegada(+60d)/pagamento(+270d); **toggle off remove saídas (cmv_imas) E os 10k cases do pipeline**. Default (sem config salva) = embarque 08/06/2026 ativo. Qtd/USD/câmbio/nacionalização ainda hardcoded (não pedidos). `ima_overdue=False`, próximo pedido dinâmico via `_ima_next_order_dt = pipeline_end − lead_time`.
- **Impostos mensais**: blended rate Aviation 11% (simples) + XConnect ~19.73% (LP) = 16.67% efetivo
- **ADS mensal**: query em `ml_ad_spend` WHERE year_month = mês atual (Aviation + XConnect)
- **CMV base**: reposição de corpos injetados lançada como saída de caixa no mês correspondente
- **Bling contatos**: campo `situacao: "A"` obrigatório no POST /contatos (fix mar/2026)

#### Imãs — Gestão de Custo por Origem (abr/2026)
- **Tabela `ima_prices`**: preço por unidade de imã por mm_size/origin/variant/effective_from, units_per_case=4
- **`product_costs.ima_prices_id`**: FK para ima_prices no componente Imãs (component_id=121) dos produtos unificados
- **Produtos unificados**: um produto por tipo (sem "Origem Brasil/China"), dois effective_from (Brasil=2000-01-01, China=data real)
- **66mm China effective_from=2026-03-06**: data real de esgotamento FIFO (IM-66-F esgotou no OP-0012)
- **88mm China ATIVO desde 2026-01-01** (M e F): preço médio ponderado R$ 31,76 (2.876 unid Brasil + 18.000 novas China). Brasil 88mm permanece como referência histórica (effective_from 2000-01-01). Saldos correntes: consultar tabela em tempo real, não confiar em snapshots
- **check-ima-depletion** (scheduler 08h15): FIFO automático sobre aba "Saídas 2026"
- **Endpoints**: `GET /api/ima-prices`, `POST /api/ima-prices` (propaga=true), `POST /api/ima-prices/propagate`
- **Planilha imãs** Drive ID `1a0G6lDQjMc_B4sIOcVT3iKwZtXKl_mTU`: abas Fechamento 2025, Saldos Iniciais 2026, Entradas 2026, Saídas 2026, Estoque Atual 2026

#### Rentabilidade por pedido (fix mai/2026)
- Ver [Rentabilidade — modelo e rateio](ecommerce_rentabilidade_pedido.md) — fórmula, NaN traps no cost_override, comissao_vendedor separada de comissao_ml, rateio proporcional ICMS/DIFAL/frete por invoice_id e shipment_id, jobs do scheduler
- Ver [ROI por vendedor](ecommerce_vendor_roi.md) — vendor-roi-report com until_ym (default mês anterior), filtro RH via COMPETENCIA (ANO=1899 bug Excel)

#### Rentabilidade / CMV (fix mar/2026) — agent/api.py + agent/metrics.py
- **Bug corrigido**: item_cost_map query usava `mi.sku = pc.sku` que fazia cross-join com SKU vazio → todos XConnect com CMV = R$9.237,87. Fix: `mi.sku IS NOT NULL AND mi.sku != ''`
- **product_costs enriquecido**: `_load_product_costs_with_map` agora também adiciona item_id→custo via ml_item_catalog (permite lookup por MLB item ID em compute_profitability)
- **Aviation 6 items**: MLB6132547098/4454360891/4454270457/4454347973/4454434195/6213931678 mapeados para "Case Injetado 66mm Aviation" com custo versionado: 2025→R$175,44 (Brasil), 2026→R$114,16 (China Marítimo)
- **Reports scheduler**: incluem seção "Outros Canais" (Bling NF-e + NFS-e + B2B manual) + Total Geral
- **Catch-up ao reiniciar**: scheduler roda report perdido 90s após subir (daily 09h + intraday 12/15/18/21h)
- **scheduler_state** tabela no DB: rastreia last_sent por job para evitar envios duplicados após restart

#### NFS-e Aviation (mar/2026)
- Bling NFS-e situacao=1 = emitida, situacao=3 = anulada (NÃO contar como receita). Filtro correto: `situacao = 1` apenas
- NFS-e incluída nos relatórios diário, semanal e mensal (seção "Serviços (NFS-e)" separada da NF-e)
- [NFs excluídas — ajustes/favores Aviation](ecommerce_nfse_exclusions.md) — NFS-e 7/8 Gustavo Oliveira (ajuste) + NF-e 81-87 maio/2026 (favor pessoal, R$ 10.842) — `exclude_from_revenue=1`. Padrão de query `AND (exclude_from_revenue IS NULL OR =0)` — 4 queries faltavam o filtro até 25/mai/2026.

#### money_release_date ML (mai/2026)
- [Provisão sem antecipação — money_release_date real](ecommerce_money_release_date.md) — CONCLUÍDO 01/06/2026: provisão usa data real de liberação do ML; XConnect líquido de 25% (empréstimo); valida vs painel "a liberar" (16.966/25.655). Fixes: wipe do release_date no sync + split real vs projeção. Frontend ProvisaoCard adaptado.
- [ML Payouts — saques MP→banco via Release Report](ecommerce_mp_payouts.md) — jun/2026: leitura automática dos saques (job payouts-sync 07h + /api/ml-payouts + conciliação c/ planilha). Quirks da API do release report do MP. Valida vs painel (XC 32/68.940, AV 17/38.460).
- [Provisão — dupla contagem + pontos cegos](ecommerce_provisao_pontos_cegos.md) — jun/2026: (a) abate `ml_ja_creditado_hoje` (estava só no response, nunca subtraído); (b1) inclui pending VENCIDO bucketizado em hoje (XC +3.032/AV +5.111); (b2) saldo MP disponível via BALANCE_AMOUNT do Release Report (tabela `ml_account_balance`, job payouts-sync, XC 548,91/AV 1.860,41). Campos novos `saldo_mp_disponivel`/`saldo_atual_total`. Testes 8/8.
- [Provisão — saldo MP zerado + data de liberação congelada](ecommerce_provisao_release_staleness.md) — 08/jun/2026: (1) `saldo_mp_disponivel` era lixo (BALANCE_AMOUNT de CSV segmentado; API direta forbidden) e foi ZERADO — MP é sacado todo fim de dia pro Inter, não é caixa de partida; (2) data de liberação preliminar (~D+28) congelava porque sync só re-checava pending VENCIDO → buraco falso de ~10 dias; fix re-consulta TODOS os pending. Bate 1:1 com painel XConnect.
- [Alertas Telegram — "enviado" era falso positivo (RESOLVIDO jun/2026)](ecommerce_alerta_enviado_falso.md) — `_run_intraday_update`/`_run_daily_sales_report` logavam "enviado" sem checar o sendMessage. Outage de DNS 02/06 17:42→03/06 07:18 fez 18/21/23h logarem "enviado" sem entregar (último real foi 15h). **Fix**: só marca `*_last_sent` se `_notify` retornar True; novo loop `retry-missed-reports` (10 min) reenvia na volta da rede e suprime intraday defasado.

#### Estoque B2B XConnect (mai/2026)
- [Planilha Estoque B2B XConnect](ecommerce_b2b_inventory.md) — Sheet `1XpSzBckN8WLO1abUrTD1syELCOHQmkL1OlRLZi__jpY` (separada dos imãs) p/ produtos vendidos por vendedor interno (não ML). Estações DC 40kW/80kW DIGITALLI. Snapshots retroativos 2026-03-31 (R$ 55.224,06) e 2026-04-30 (R$ 75.915,63) inseridos como `tipo='b2b'` em `estoque_snapshots`. **CONCLUÍDO (verificado jun/2026)**: reader `_fetch_b2b_inventory_raw` (api.py:8408), `GET /api/b2b-inventory` (api.py:8542), seção no `estoque/index.tsx:1060`, config `B2B_INVENTORY_SHEET_ID` (config.py:64). Bot Telegram aceita PDF com caption/reply `NF ENTRADA` → salva em `/home/ubuntu/nfs_entrada/`.

### 3. b3analyzer (`/home/ubuntu/b3analyzer`)
- CLI de análise do mercado B3 (ações, opções, fundamentalista, técnica)
- Python 3.12, Click, Rich, pandas, yfinance, scipy, streamlit, plotly
- 17 comandos, Dashboard Streamlit porta 8501, alertas Telegram
- **Crawl4AI**: web scraping de InfoMoney + Valor Econômico (data/crawl_client.py)
- **ChromaDB**: vector store local em ~/.b3analyzer/chromadb/ (data/vector_store.py)
  - Coleção `financial_news_v2` com hash embedding próprio (evita sentence-transformers)
  - SEGFAULT corrigido: ChromaDB+FinBERT conflitavam no OpenBLAS → substituído por `_hash_embed`
- **RAG Engine**: narrativas com Claude Haiku via ChromaDB context (analysis/rag_engine.py)
- **b3-alerts daemon**: auto-restart por RSS>1GB é normal (FinBERT usa ~800MB)
- Ver detalhes em `b3analyzer_details.md`

### 4. dividend-portfolio (`/home/ubuntu/dividend-portfolio`)
- Dividend Growth Portfolio Engine — screening, Moat/Munger, Kelly Criterion, Monte Carlo
- Python 3.12, Click CLI (`div`), Rich, yfinance, Streamlit (porta 8506)
- Arquitetura espelha b3analyzer: SQLite cache TTL, Telegram alerts, loguru
- Config dir: `~/.dividend-portfolio/`, cache: `cache.db`, portfolio: `portfolio.json`
- 6 comandos: screen, analyze, portfolio, simulate, monitor, config
- 6 pages Streamlit: overview, screening, analysis, simulation, dividends, rebalancing
- Fontes: yfinance (preços/dividendos/fundaments), FRED API (macro BCB/Fed)
- yfinance DY para B3 é unreliable — calculamos DY de dividendos reais 12m

### 5. followup-agent (`/home/ubuntu/followup-agent`)
- Gestão de tarefas/follow-ups via Telegram com linguagem natural + voz
- Python 3.12, Telegram Bot API, Claude Haiku (NLU), Groq Whisper (áudio)
- **Prompt Caching**: system prompts com cache_control ephemeral (nlu.py)
- **Mem0**: memória persistente local (memory.py)
- 5 categorias, Lembretes automáticos 09h/14h/18h
- Estrutura: agent.py, nlu.py, store.py, telegram.py, transcriber.py, dashboard.py

#### Empréstimo ML XConnect (abr/2026)
- Ver [ecommerce_emprestimo_ml.md](ecommerce_emprestimo_ml.md) — contratado 10/04/2026, R$214k recebido, R$246.027,99 a pagar, 25% retenção diária ML XConnect

## Próximos Passos
- [Havan — incrementos pendentes](ecommerce_havan_pendencias.md) — **adiado pelo usuário jun/2026** (5 features principais concluídas). Whitespace por filial, dimensão comprador, elasticidade de preço, digest Telegram. Revisar quando ele perguntar "o que tem pendente de implementação".
- **Pendente — Validar dados Brasil — DRE**: página `"Brasil — DRE"` implementada em `app.py` (mai/2026). Receita = BQ fee_liquido+setup por competência; Despesas = planilha por sub_centro/competência. Precisa revisão manual dos números antes de usar em decisões: conferir se categorias sub_centro (MÍDIA, RH, COMERCIAL BR, etc.) batem com o que está na planilha, e se o resultado operacional faz sentido com o que se conhece do negócio.
- [BQ Brasil × Intl — estado real](email_bq_brasil_intl_integration.md) — `get_unified_billing` já cruza Brasil+LATAM em BRL (Executive, Global Ranking, FastAPI). Gap real é projeção: Brasil ainda fora de CashFlowProjection/Runway/P&L. 3 caminhos no doc.
- [Brasil — pipeline projeção C revisado](email_brasil_proj_pipeline.md) — `build_fluxo_brasil_proj` em `cashflow_daily_reader.py`: Vcto + delay calculado do BQ p/ atípicos + replicação forward. 3º modo na página Fluxo de Caixa Diário. +85% entradas em 60d.
- **Grupos sem cadastro Brasil — RESOLVIDO 10/mai/2026**: `agent/financeiro/grupos_override.json` com **3.063 CNPJs / 96 grupos**. Fonte primária: planilha "RANKING MOTORLEADS 2026" (ID `1Px7mCwhDQtjU_CGmnqSUdz3pZVrGxbnho6Si9CKBECU`), abas FATURAMENTO JAN/FEV/MAR 2026 Brasil → 176 CNPJs. Residuais (34): regras automáticas. **Regra HAGAH**: qualquer CNPJ com `*` em `Cliente` ou `Nome_Fantasia` do BQ → grupo HAGAH; aplicado a 2.910 CNPJs históricos (R$ 4,84M receita histórica, maior parte de 2016-2019). Conflitos de capitalização consolidados (HAGAH/Hagah, KAIZEN/Kaizen, HAI/Hai). Internacional: `nome_fantasia` BQ já é 1:1 com Grupo, não precisa override. Backups: `.bak-pre-176`, `.bak-pre-residuais`, `.bak-pre-hagah-historico`.
- **Brasil — Atendimento — B+D + MRR + upsell expandido aplicado 10/mai/2026**: Painel de Ação (Tab 1) agora tem filtro por vendedor + filtro por grupo + coluna Grupo + coluna **MRR Recuperação (R$)** (ganho mensal real da ação). Helper `load_brasil_metadata_recent` em `app.py:670+` (lookup CPF→{vendedor, grupo, estado}, cache 1h). Vendedor: 28 distintos, 100% cobertura. Grupos: 215 distintos. **3 tipos de upsell** agora: Facebook (Google sem FB), **TikTok (FB≥1k sem TikTok)**, **Waze (fee≥2k sem Waze, com mídia ativa)**. Volume: ~67 oportunidades vs 30 antes. KPIs: 5 colunas, MRR Recuperação total. Ordenação: urgência → grupo → MRR Recuperação. **Nota:** Carlos Eduardo Castro Amado é CFO, clientes "dele" são órfãos (68% HAGAH). Doc: `docs/auditoria_brasil_atendimento.md`. Sub-melhorias pendentes: histórico de tentativas (refator de componente React), SLA por urgência.
- **Pendente — Brasil no fluxo de caixa diário com lógica de projeção**: hoje só Internacional usa `recv_detail_all` da `CashFlowProjection`. O Brasil ainda usa apenas vencimento BQ. Quando integrar payment_terms para clientes Brasil, replicar `fetch_entradas_from_projection` para o tab Brasil.
- [Resultado por País — revisão pendente](email_resultado_pais_revisao.md) — página mistura accrual+cash, semântica de Dívidas Pagas vs Prev., e sobreposição com Fluxo de Caixa/DRE/P&L. Ajustes técnicos aplicados em mai/2026; revisão conceitual em outra sessão.

## CAC Brasil — resolvido (2026-04-28)
- Fonte de despesas: planilha consolidada Google Sheets ID `1YTO_5sJ0m48i-2b-xdOn3OPioMQ8Pi1_dUNjFLeYdwM`, aba `Despesas`
- Reader: `agent/financeiro/despesas_brasil_reader.py`
- CAC numerador = `PACOTE NOME='Despesas com Vendas Comerciais'` + `SUB CENTRO='COMERCIAL BR'`
- MÍDIA = verba de mídia dos clientes (pass-through, NÃO entra no CAC)
- B1 resolvido, flag em `docs/b1_resolved.flag`, lembrete parado

- [OAuth tokens ecommerce-agent](ecommerce_oauth_tokens.md) — token_sheets.json (spreadsheets) + token_drive.json (drive.readonly); NUNCA misturar scopes. Re-auth Drive via `scripts/setup_drive_oauth.py` (flow manual remoto). Tokens podem revogar mesmo em production — guard com alerta no `_fetch_imas_data`.
- [Cases — capacidade por tipo de imã](ecommerce_cases_capacidade.md) — F e M são modelos de case diferentes, somam na capacidade total. NUNCA tomar min(F,M) como gargalo.
- [Runway de cases — métricas duais](ecommerce_runway_metricas.md) — `pipeline_months` (ritmo atual, 30d) ≠ `pipeline_months_pico` (pior caso, max trend). Trigger usa pico, display mostra ambos. Tendência inclui mês corrente normalizado.
- [ml_nfe gotchas](ecommerce_ml_nfe_data_emissao.md) — DOIS bugs: ~30-50% das NFs sem data_emissao + linhas duplicadas quando ML consolida pedidos. Sempre JOIN com ml_orders_sync por `o.data` e GROUP BY `n.invoice_key`. Upsert preserva campos via COALESCE; re-sync forçado após 6h.
- [Projeção ML cashflow](ecommerce_projecao_ml.md) — default `ma_30d` (não weighted_3m). cur_estimate sem blend (extrapolação pura). Impostos (loop principal + simulação parcelamento + _is_next_mo) e ADS escalam com base BRUTA projetada (`ml_monthly_avg_gross_*`) — não mês anterior fixo. ICMS escala proporcional.

## Preferências do Usuário
- [Idioma das respostas](feedback_language.md) — sempre escrever em português
- [Análises mensais até fim do mês](feedback_final_mes.md) — usar sempre último dia real do mês (28/29/30/31), nunca 30 fixo
- [Assets Google do ecommerce-agent](feedback_ecommerce_google_assets.md) — toda planilha/Drive do serviço deve ser criada em xconnectimport@gmail.com, nunca na conta pessoal

## Monitoramento Cross-Project
- [Claude Usage Dashboard](claude_usage_dashboard.md) — pacote `claude-usage` + dashboard Streamlit (porta 8520) que rastreia tokens/custo Anthropic em todos os 5 projetos. Wrapper integrado em email/ecommerce/followup/b3/dividend. SQLite em `~/.claude-usage/usage.db`.

## Padrões Técnicos
- Anthropic SDK 0.84.0, prompt caching: `cache_control: {"type": "ephemeral"}`
- Streaming: `async with client.messages.stream() as stream: async for text in stream.text_stream:`
- Extended thinking (estilo antigo): `thinking={"type": "enabled", "budget_tokens": N}`, max_tokens > budget_tokens
- Adaptive thinking (Sonnet 4.6 / Opus 4.7): `thinking={"type": "adaptive"}` + `output_config={"effort": "low|medium|high|max"}`
- [Adaptive thinking + tool_use exige signature](adaptive_thinking_signature.md) — thinking blocks devem voltar com `signature` no loop tool_use; não persistir thinking em storage de longo prazo
- Telegram streaming: sendMessage → message_id → editMessageText a cada 0.8s
- Todas features com try/except fallback para comportamento original
- [Empréstimo Sicredi na provisão](ecommerce_sicredi_emprestimo.md) — parcelas SAC + Selic mês a mês do BCB; RF em garantia bloqueada; custo efetivo c/ garantia; **IOF auto-detectado do Sheets** (15/jun) somado ao custo da operação
- [Paridade provisão × gráfico](ecommerce_provisao_grafico_paridade.md) — tudo na provisão precisa aparecer no gráfico do fluxo (CFO)
