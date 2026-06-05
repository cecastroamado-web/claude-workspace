---
name: email-wire-intercompany
description: Detecção automática de WIRE intercompany (Motorleads ↔ Motorleads) em banco_reader; corrige fluxo líquido por país que somava transferência interna como receita/despesa
metadata: 
  node_type: memory
  type: project
  originSessionId: 0d42e0cf-f4ec-4094-b3f8-6dfe2fd5b376
---

WIRE entre subsidiárias Motorleads (ex.: Reweb Peru → Reweb Colombia) é detectada automaticamente pelo beneficiário/descrição da transação bancária e marcada como `intercompany=True` na `BankTransaction`. Implementado em mai/2026.

**Why:** Antes, uma wire intercompany inflava `wire_in` no país receptor e `wire_out` no remetente. Isso distorcia o fluxo líquido por país (Colômbia parecia melhor, Peru pior), embora o consolidado fosse neutro. EOM projetado a nível empresa NUNCA foi impactado (saldo total preserva).

**How to apply:**
- **Regra de detecção** (`agent/financeiro/banco_reader.py` → `_is_intercompany`): texto normalizado (benef + desc) contém `motorleads` + qualquer um de `colombia|peru|mexico|ecuador|equador|chile|argentina|brasil|brazil`. Aplicado em `read_banco_tab` e `_read_brasil_banco`. Padrões reais encontrados nos extratos: `"Wire Colômbia (Fatura N de Motorleads)"` (Peru OUT) + `"Empréstimo recebido de Reweb Peru ... (Fatura N de Motorleads)"` (Colombia IN).
- **Buckets no agg** (`aggregate_monthly`): `wire_in_intercompany` / `wire_out_intercompany` acumulam o subconjunto intercompany; `wire_in` / `wire_out` continuam totais. Consumidores subtraem para fluxo operacional puro.
- **Pontos corrigidos**:
  - `agent/dashboard/app.py:1624` — "Resultado por País" → tab Fluxo de Caixa: fluxo líquido exclui intercompany; tabela ganha coluna "Intercompany".
  - `agent/financeiro/projection.py:445` — `real_wire_in` (→ `proj.wire_in_month`) ignora `tx.intercompany`. Sem isso, o waterfall `chart_projection_waterfall` em `charts.py:734` adicionava a wire intercompany como entrada relativa no mês (caso real 14/mai: COL recebeu 7,1M COP ≈ USD 1,9k de PER, aparecia como barra verde "WIRE In" $1,9k mesmo já estando no Saldo Atual).
  - `agent/financeiro/projection.py:478` — média histórica de WIRE OUT (M-1, usada para projetar M+1) exclui intercompany; senão a wire intercompany de meses passados inflava `wire_expected_usd`.
  - `agent/financeiro/cashflow_daily_reader.py:1785` — fallback (sem `media_monthly`) do `fetch_wire_out_internacional` também filtra intercompany na média 3m do banco.
- **Não corrigidas (intencionalmente)**: página WIRE (`app.py:3262`), tabelas Fluxo de Caixa consolidado, `chart_wire_bars`. Mostram tudo para auditoria. Se for incomodar, consumir `wire_in_intercompany` / `wire_out_intercompany` do agg.
- **Validação 10/mai**: 10 transações detectadas em 12m (5 pares Peru↔Colombia ~USD 9-10k cada). Para auditar: `read_all_banco(...); for tx in ...: if tx.intercompany: ...`.
- **Para adicionar nova subsidiária**: appendar token de país em `_INTERCOMPANY_COUNTRY_TOKENS` (já normalizado, sem acento). Padrão exige "motorleads" presente — se uma operação intercompany não passar por entidade Motorleads, ajustar regex.
- Relacionado: [[email-resultado-pais-revisao]] (a página principal afetada).
