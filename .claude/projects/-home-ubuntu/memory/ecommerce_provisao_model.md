---
name: Provisão de Caixa — Modelo de Antecipação ML
description: "[SUPERSEDED desde 20/05/2026] modelo ANTIGO de antecipação ML — substituído pelo modelo sem antecipação em [[ecommerce-money-release-date]]"
type: project
originSessionId: 007a3951-8cee-4ab9-911f-651c3ee57166
---
> ⚠️ **SUPERSEDED (20/05/2026).** Não há mais antecipação. A provisão (`/api/provisao` E o alerta Telegram `_run_check_provisao`) agora entra com o caixa na **money_release_date REAL do Mercado Pago**. Use [[ecommerce-money-release-date]] como verdade atual; o que segue abaixo é histórico. NÃO reintroduzir `disponivel_ant_hoje`/média diária/custo 4,16% na projeção.

> 🔧 **Refactor jun/2026 (verificado e funcionando):** `_run_check_provisao` (scheduler, diário 08h) foi reescrito (−167 linhas) para **consumir `get_provisao(empresa, dias=30)` direto** como fonte única de verdade, em vez de reconstruir a lógica lendo Sheets (`get_a_receber_por_dia`/`get_contas_pagar_por_dia`/`get_bank_balance`). Limiar amarelo R$20k vem do próprio `/api/provisao`; constante morta `PROVISAO_SALDO_ALERTA` removida. Alerta mostra saldo total (banco + MP a sacar), releases reais + projeção 30d, e os dias em déficit.

## Modelo de Antecipação ML (confirmado abr/2026 — OBSOLETO)

O usuário antecipa TODOS os dias no ML. Isso define o modelo correto para a projeção de caixa:

- **Antecipação é no mesmo dia (D-0)**: as vendas líquidas de hoje podem ser antecipadas hoje mesmo, não no dia seguinte
- **Tudo até ontem já foi antecipado**: o saldo_atual já reflete todas as antecipações até ontem — não projetar releases D+14 de ordens passadas (causaria double-counting)
- **Taxa**: 4,16% flat sobre valor líquido (gross - sale_fee - shipping_cost - financing_fee)

## Lógica da projeção (`/api/provisao`)

```
disponivel_antecipacao_hoje = vendas líquidas de HOJE no DB (parcial, cresce ao longo do dia)
media_diaria_ml = net_14d_completos / 14  (últimos 14 dias fechados, exclui hoje que é parcial)

releases_por_dia:
  today → disponivel_antecipacao_hoje (real, estimado=False)
  tomorrow+ → media_diaria_ml (estimativa, estimado=True)
```

**NÃO usar** `get_pending_releases()` (D+14 lookback) — isso conta dinheiro que já está no saldo.

## Thresholds semáforo
- Verde: saldo projetado ≥ R$50.000
- Amarelo: R$20.000 – R$49.999
- Vermelho: < R$20.000

## Entradas além do ML
- A RECEBER da planilha (títulos B2B/Shopify, STATUS='A RECEBER') incluídos como inflows diários
- Aviation tem ~5 títulos nos próximos 30 dias (BRUNO ROCHA, SEMPRE BOM, STAR2GO, SOTRIMA)

**Why:** O modelo D+14 de releases passados causava double-counting porque o usuário já antecipou tudo. A lógica correta é: hoje real + média histórica para o futuro.

**How to apply:** Sempre que mexer em `/api/provisao` ou `_run_check_provisao`, manter esse modelo. Não reintroduzir `get_pending_releases()` na projeção.
