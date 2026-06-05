---
name: ml-payouts-release-report-do-mercado-pago
description: ecommerce-agent — leitura automática dos saques MP→banco via Release Report; quirks da API e conciliação com a planilha
metadata: 
  node_type: memory
  type: project
  originSessionId: e0197d9a-df43-4b31-a9f7-e1b367bd8a70
---

ecommerce-agent: leitura automática dos **saques (payout)** do Mercado Pago (dinheiro transferido do MP pro banco), implementada jun/2026 (commit a0f07dd).

## Por que via Release Report
O token OAuth MLB **não** tem escopo pra saldo/movimentos direto: `GET /users/{id}/mercadopago_account/balance` → 403; `/v1/account/movements`,`/withdrawals` → 404. O que funciona é o **Release Report** (relatório assíncrono de liberações), que tem TODOS os movimentos da conta MP.

## Fluxo da API (quirks importantes)
1. **Gerar**: `POST {MP_BASE_URL}/v1/account/release_report` body `{"begin_date":"YYYY-MM-DDT00:00:00Z","end_date":"...Z"}` — formato com `Z` **SEM milissegundos** (com `.000Z` dá 400 "invalid_begin_date"; body, não query). Retorna 202. Geração demora de 1 a ~20+ min.
2. **Listar**: `GET .../release_report/list` → cada item tem `status` (`'enabled'`=pronto), `file_name`, `begin_date/end_date`. **Não confiar em `download_date`** — só marca "já baixado" (fica null mesmo com o relatório pronto). Use `status=='enabled'`.
3. **Baixar**: `GET .../release_report/{file_name}` → CSV separador **`;`**. Colunas: DATE;SOURCE_ID;DESCRIPTION;NET_CREDIT_AMOUNT;NET_DEBIT_AMOUNT;...;BALANCE_AMOUNT.
4. **Saques** = linhas `DESCRIPTION=='payout'`, valor=`NET_DEBIT_AMOUNT`, id único=`SOURCE_ID`. Outros DESCRIPTION úteis: `payment` (vendas liberadas), `reserve_for_debt_payment` (NÃO é a retenção do empréstimo do sócio), `fee-release_in_advance` (taxa antecipação), `mediation`/`refund`/`reserve_for_dispute`.

## Componentes (commit a0f07dd)
- `mercadolivre.py`: `generate_release_report`, `list_release_reports`, `fetch_payouts(begin,end)` (acha enabled que cobre a janela, baixa, parseia; retorna [] se nenhum enabled — NÃO espera).
- `db.py`: tabela `ml_account_payouts` (UNIQUE company+source_id) + `upsert_payouts`/`get_payouts`.
- `scheduler.py`: job `payouts-sync` (1×/dia 07h + catch-up): gera relatório de 45d, baixa o enabled mais recente, upsert. Como a geração é assíncrona, o relatório pedido hoje é consumido num run seguinte.
- `api.py`: `GET /api/ml-payouts?empresa=&date_from=&date_to=` → payouts + `reconciliacao` por dia (MP vs `get_ml_creditado_por_dia` da planilha) + total_mp/total_planilha. Fallback on-demand se tabela vazia.

## Validação (mai/2026)
XConnect 32 payouts R$ 68.940 · Aviation 17 R$ 38.460 — batem com o painel ML e com os créditos diários da planilha. A reconciliação evidencia 30/04 (61.901) e 31/05 (36.225) como **retenção do empréstimo** (par venda+EMPRESTIMO net-zero), NÃO payout — ver [[ecommerce_money_release_date]]. A retenção do empréstimo é IMPLÍCITA (liberação menor), não aparece como movimento avulso no relatório.

## Rodar scripts que importam agent.* fora do serviço
Sempre com cwd em `/home/ubuntu/ecommerce-agent` E via heredoc/stdin, OU `PYTHONPATH=/home/ubuntu/ecommerce-agent` — senão importa o pacote `agent` de site-packages (exige ANTHROPIC_API_KEY e quebra).
