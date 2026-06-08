---
name: email-projection-filtros
description: "Filtros e parâmetros centrais da projeção mensal Internacional do email-agent — limite de atraso, override Irrecuperável via recv_state, include_overdue"
metadata: 
  node_type: memory
  type: project
  originSessionId: 70ac2e93-4aea-4912-9e9f-c218bfcec831
---

`agent/financeiro/projection.py` — projeção mensal Internacional. Comportamentos não-óbvios:

**Filtro de PDD efetivo** (`projection.py:157`): faturas com `(today - expected).days > 120` são descartadas do `recv_detail_all`. Foi **60 → 120 dias** em 2026-05-18 a pedido do usuário, porque o limite anterior fazia faturas vencidas há 60-120 dias sumirem do painel "Em Atraso" do dashboard sem permitir marcação manual como Irrecuperável.

**Why:** o usuário precisa enxergar faturas vencidas no painel para marcá-las como "Irrecuperável" via `recv_state.json`. Se o filtro derruba antes, a UI não tem o que mostrar.

**How to apply:** Se subir mais o limite, lembre que faturas muito antigas inflam recv_month indevidamente quando o modo "Adicionar Hoje/EOM" é ativado. Mantenha o limite menor que ~180d.

**Override Irrecuperável + Promessa** (`app.py:load_projection`): lê `agent/dashboard/recv_state.json`. Para faturas com `status_cobranza == "Irrecuperável"` marca `inv.juridico=True`. Para `status_cobranza == "Promessa"` com `data_promessa` válida, preenche `inv.data_prometida` (entra na projeção pela data prometida via lógica padrão de `projection.py:122`). Semântica é **por fatura individual** (match por `(invoice_number, client)`).

**UI Promessa+data** (`receivables_table/src/ReceivablesTable.tsx`, build em May/2026): nova coluna "Data Promessa" com input `type="date"` visível só quando `status_cobranza === "Promessa"`. Persistido em `recv_state.data_promessa` (string ISO YYYY-MM-DD). Rebuild via `cd agent/dashboard/components/receivables_table && npm run build`.

**Why:** o dashboard salva status_cobranza por fatura via componente do recv_records. A `projection.py` original não consultava esse arquivo (só `juridico.json`, que é lista global de clientes em PDD).

**How to apply:** Se o usuário quiser "marcar cliente todo como PDD", ele pode: (a) marcar cada fatura no dash; (b) adicionar o nome no `/home/ubuntu/email-agent/juridico.json` (lista global). Pendente: decidir se vale fazer auto-promoção (marcar 1 fatura = todas).

**include_overdue (radio na página Projeção)** — implementado 2026-05-18. Param `include_overdue: "skip"|"today"|"eom"` em `build_cash_flow_projection`. Quando != "skip", vencidos sem `data_prometida` com `expected < month_start` têm expected realocado para today/EOM e marcam `source="vencido (forçado)"`. Propaga pelos 3 tabs (Intl/Brasil/Cons) via radio horizontal no topo da página.

`data_prometida = "Não Cobrar"` (string literal, convenção CFO) NÃO é tratado como promessa válida — cai no else por causa de `_parse_date()` falhar. Mas a string é truthy, então `not inv.data_prometida` é False → NÃO é forçado. Comportamento correto: fica em recv_month via histórico.

**Antecipação robusta no clamp** (`projection.py:~160`, decisão CFO 2026-06-06): para fatura ainda não vencida com expected histórico < due, o clamp ao vencimento é PULADO se o perfil tem `count >= 10` e `avg <= -3` dias (expected floor = today). **Why:** ~45 faturas/mês vencem dia 01 do mês seguinte (formalidade net-30; maio venceu 31/05 por ter 31 dias) mas 33 delas são de dealers que pagam dentro do mês de competência (avg −4 a −23d com 11-52 amostras). O clamp antigo empurrava tudo para julho → recv_service_month caiu de ~38k para 17,6k em jun/2026 e o CFO estranhou. Com a regra: 33,9k. **How to apply:** não baixar a régua (≥10/−3d) sem pedido — perfis com poucas amostras têm avg negativo por calibragem antiga. Captions "🔜 Fora da janela do mês" nos 3 tabs da página Projeção mostram o cliff dos 7 primeiros dias do mês seguinte.
