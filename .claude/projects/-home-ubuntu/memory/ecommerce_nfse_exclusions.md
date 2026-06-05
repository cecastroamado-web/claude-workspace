---
name: NFs Exclusões Aviation — Ajustes Contábeis e Favores
description: NF-e e NFS-e da Aviation que não devem ser consideradas receita real. Marcadas com exclude_from_revenue=1 no DB.
type: project
originSessionId: 97dbc506-9191-42ac-813d-b6e4135ca9cd
---
## Como funciona o opt-out de receita

Tabelas `bling_nfe` e `bling_nfse` têm a coluna `exclude_from_revenue` (BOOLEAN, default 0). Quando setada para 1, a NF é ignorada por queries de receita, alertas, cobranças, relatórios e DRE.

**Padrão de query**: `AND (exclude_from_revenue IS NULL OR exclude_from_revenue=0)` — usar em TODA query de revenue. Histórico mostra que é fácil esquecer; em 25/mai/2026 corrigi 4 queries (bling-revenue, profitability, alerts, AI analyze). Em 01/jun/2026 corrigi mais 3 em `scheduler.py`: alerta `check-nfe-sem-custo` (~L1088), alerta `check-nfe-custo` (~L2503) e a query do relatório de rentabilidade/faturamento diário/semanal/mensal (~L1853) — as NF 81-87 ainda apareciam nos alertas "custo não vinculado" do Telegram e como faturamento.

**EXCEÇÃO — imposto fica na base:** as queries de BASE DE IMPOSTO da provisão (`api.py` ~14467/14487, DAS Aviation) e a auditoria do contador (`api.py` ~4958) **incluem** as notas excluídas de propósito. Para favores, o imposto (DAS) é devido e pago — **o amigo reembolsa o imposto** — então permanece na provisão de caixa. exclude_from_revenue tira do FATURAMENTO gerencial, não da obrigação fiscal.

Para marcar uma NF nova como excluída: `UPDATE bling_nfe SET exclude_from_revenue=1 WHERE company='...' AND CAST(numero AS INTEGER) IN (...)`.

## NFS-e excluídas (`bling_nfse`)

### GUSTAVO OLIVEIRA DE MACEDO (41.259.919) — Aviation
- **NFS-e 7** — 29/11/2025 — R$194.849,00 — situacao=1 (emitida)
- **NFS-e 8** — 12/12/2025 — R$194.849,00 — situacao=3 (anulada)

**Por quê:** Ajustes contábeis, não são receita real e não serão pagas.

## NF-e excluídas (`bling_nfe`)

### Favor pessoal — Aviation, 25/mai/2026
NFs **81 a 87** (sit=6) totalizando R$ 10.842,05. Emitidas como favor para um amigo, não são receita real.
- 81: Vitor Komaroff Simoes — R$ 449,00 (CFOP 6108)
- 82: St Servicos Empresariais — R$ 29,00 (5102)
- 83: Hugo Costa Gomes — R$ 1.338,00 (6108)
- 84: Aparecido Samartino Jr — R$ 1.044,05 (6108)
- 85: Ingrid Cristina Armenini — R$ 1.987,00 (6108)
- 86: Thiago Marques — R$ 2.098,00 (6108)
- 87: Strata Engenharia — R$ 3.897,00 (6102)

**How to apply:** Ignorar em qualquer contexto de receita, cobranças, alertas, relatórios e planilha. Não perguntar se foram recebidas. Quando aparecerem em logs, lembrar que são favor (não bug). Se o usuário emitir novas NFs do mesmo tipo, perguntar números e marcar `exclude_from_revenue=1`.
