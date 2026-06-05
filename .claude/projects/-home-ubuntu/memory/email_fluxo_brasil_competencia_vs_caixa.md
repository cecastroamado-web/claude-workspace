---
name: Brasil — competência vs caixa no Fluxo Diário
description: Despesas PROG da planilha estão por competência; algumas serão parceladas/renegociadas e não pagas na data, financiando a operação via imposto
type: project
originSessionId: c738a5e8-9708-47a0-9874-623b8cc12be8
---
Pendência aberta (mai/2026) — regras já implementadas + decisão pendente sobre parcelamento:

## Implementado

`classify_rigidez()` em `cashflow_daily_reader.py` classifica cada PROG em 4 níveis:
- **RIGIDO** — folha/telecom/mídia ao vivo (não dá pra atrasar)
- **FLEXIVEL** — fornecedores, jurídico, **parcelamentos JÁ em curso** (MENSAL=NAO RECORRENTE — precisam cumprir)
- **PARCELAVEL** — impostos da competência corrente (MENSAL=RECORRENTE + Min. Fazenda/INSS/ISS/Prefeitura) — candidatos a parcelamento futuro
- **OPCIONAL** — CAPEX, viagens, pró-labore

Dashboard:
- Radio "Cenário caixa Brasil" no Fluxo Diário (Conservador/Realista/Otimista)
- Expander "🏛 Tributários parceláveis pendentes" lista os PARCELÁVEIS automaticamente
- Build_cash_flow_projection (Executive Overview) consolida Brasil PROG conservador como despesa default

## Decisão pendente do usuário (mai/2026)

Listados R$ 50.277 de tributários parceláveis pendentes em maio/2026 (8 lançamentos):
- R$ 18.765 DARF Min. Fazenda — Impostos sobre Receita
- R$ 15.226 ISS Prefeitura de Osório
- R$ 13.328 INSS folha
- R$ 2.957 ISS Porto Alegre + Min. Fazenda menores

Usuário sinalizou que vai **decidir depois** quais efetivamente parcelar. Vai usar o expander como referência quando for tomar a decisão.

**How to apply:**
- Quando usuário pedir relatório do mês, citar tanto Conservador (R$ 494k total) quanto Realista (R$ 444k = -R$ 50k de parceláveis) para ele decidir
- Não recomendar parcelar automaticamente — é decisão de negócio (custo de juros do parcelamento vs benefício do caixa)
- Saídas reais via extrato bancário (banco_reader) mostram ~R$ 768k/mês operacionais nos últimos 12m — qualquer regra de fluxo Brasil tem que reconciliar com essa base
