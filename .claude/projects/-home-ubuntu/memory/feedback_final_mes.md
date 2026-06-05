---
name: Análises mensais sempre até o último dia do mês
description: Quando o usuário pede "no mês de X" ou "até fim do mês", incluir todos os dias até o último dia do mês (28/29/30/31), nunca cortar em 30 fixo
type: feedback
originSessionId: c738a5e8-9708-47a0-9874-623b8cc12be8
---
Em análises de fluxo de caixa, despesas, receitas, etc., quando o usuário se referir a um mês inteiro:
- Sempre considerar até o último dia real do mês (31 para jan/mar/mai/jul/ago/out/dez, 30 para abr/jun/set/nov, 28/29 para fev)
- Não usar dia 30 como aproximação para todos os meses
- Esse padrão aplica-se a todos os relatórios e queries do dashboard MotorLeads (e provavelmente outros projetos financeiros)

**Why:** Em maio/2026, ao validar saídas pendentes, a diferença entre "até 30/05" (R$ 475k) e "até 31/05" (R$ 493k) foi de R$ 18k em 14 lançamentos típicos de fim de mês (CAJU, parcelamentos PMPA, etc.). Cortar em 30 perde lançamentos relevantes.

**How to apply:** Em queries SQL, filtros de data e agregações mensais, usar `last_day(month)` ou `calendar.monthrange(year, month)[1]`. Nas comparações que mostro ao usuário, default = último dia do mês.
