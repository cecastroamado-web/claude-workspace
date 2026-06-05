---
name: Internacional — Critérios Financeiros Consolidados
description: Toda a auditoria e padronização das 20 páginas do dashboard email-agent que tratam dados Internacional. Validado em mai/2026.
type: project
originSessionId: 621b5cc7-1075-45f1-bd33-104fa36590ef
---

# Internacional — auditoria 100% concluída (mai/2026)

**Status**: Todas as páginas Internacional do dashboard `email-agent` foram revisadas e padronizadas.

**Doc completo**: `/home/ubuntu/email-agent/docs/criterios_financeiros_internacional.md`

## Estrutura padrão de DRE Internacional

```
Receita Serviço (sem IVA)
(-) OpEx (Recorrente + Não-Recorrente, sem dívida e sem IVA)
= Margem Operacional / Margem Líquida   ← métrica do negócio
(-) Parcelas Dívida                      ← despesa financeira (info)
= Resultado de Caixa                     ← gestão de caixa
IVA (info)                               ← pass-through, NUNCA na margem
```

**IVA é pass-through**: cliente paga + empresa repassa ao fisco. Receita registrada já é sem IVA, então subtrair IVA pago seria assimétrico. Vale para todas as páginas (DRE, P&L, Resultado por País, Comparativo MoM).

**Dívidas saem da Margem**: vão como linha separada para "Resultado de Caixa", preservando margem operacional como métrica do negócio.

## Câmbio histórico (regra absoluta)

- `to_usd_m(val, currency, year, month)` — qualquer valor com data conhecida
- `to_brl_m(val, currency, year, month)` — idem para BRL
- `to_usd(val, currency)` — APENAS saldo atual de conta (`acc.latest_balance`)
- `_inv_to_usd(inv, currency)` — helper para faturas (mes_ref → due_date → fallback)

Páginas com toggle BRL/USD: Provisão Diária BQ, Global Ranking.

## Cadeia de previsão de pagamento

```
1. data_prometida (coluna BJ ou Observação) — só ABERTOS/VENCIDO, NÃO em A FATURAR
2. payment_terms.json (pay_day/delay/billing_lag)
3. _get_pay_day_pattern (auto-detect dia fixo)
4. _get_profile.avg > 25d (avg histórico)
5. country_pay_day (mediana do país — México=16, Colombia=18, Argentina=22…)
6. Vencimento da fatura (fallback final)
```

**Bug fix**: clamp `expected >= due` só dispara para `source='histórico'` (preserva data_prometida quando cliente promete pagar antecipado).

## Sequência de pagamentos no Fluxo Diário Internacional

- Dia 15: Despesas Não-Recorrentes (média 3m)
- Dia 25: WIRE OUT (antes do vcto cartão dia 28)
- Dia 27: Dívidas
- Dia 29: Compromissos

WIRE OUT bottom-up: `mídia M-1 + max(0, serviço M-1 − despesas_locais_M-1)` por país.

## Atrasados Internacional
Cutoff de **90 dias** — acima disso é PDD/irrecuperável (visível em Provisão Diária BQ).

## Replicação de receita futura
Em horizons > 30d, `fetch_entradas_from_projection` replica receita do mês corrente nos meses M+1, M+2... preservando dia de pagamento por cliente. Dedup por cliente×mês.

## How to apply (futuras alterações)
- Ao adicionar página nova, seguir critério de `docs/criterios_financeiros_internacional.md`
- Margem operacional NUNCA inclui IVA ou dívidas
- Câmbio histórico em qualquer valor com data conhecida
- Mês corrente excluído de comparativos MoM (incompleto)

## Páginas refatoradas (16) + sem mudança (4) = 20 total

**Refatoradas**: Projeção, Fluxo Diário, DRE Competência, Compromissos, Resultado por País, P&L por País, Comparativo MoM, Fluxo de Caixa, Aging Recebíveis, Executive Overview, Recorrente, Cenários Runway, Provisão Diária BQ, Global Ranking, Burn Rate, Alertas.

**Sem mudança (já alinhadas)**: WIRE, Saldos, Fornecedores, Sazonalidade, Anomalias, Metas/Budget, Perguntas.

## Próximo passo
Iniciar auditoria de páginas Brasil seguindo mesmo padrão (DRE, OpEx sem dívida/IVA, recebíveis com cutoff). Brasil tem fontes próprias (BQ `faturamento`, planilha trimestral de despesas) — precisa mapear equivalentes de payment_terms, BJ, country_pay_day para o contexto Brasil.
