---
name: ADS projeção cashflow — fix extrapolação parcial mês
description: Bug e fix da projeção de ADS no cashflow quando poucos dias do mês têm dados
type: project
originSessionId: c2f7eae1-140e-4e95-ac10-6c321733ef0e
---
## Bug: ADS extrapolado do dia 1 inflava o valor projetado

**Sintoma**: No dia 1 de maio/2026, ADS projetado aparecia como R$14.982 (= R$483 × 31 dias) em vez de ~R$35.919 (média mês anterior).

**Causa raiz**: Dois lugares com cálculo de ADS no `agent/api.py`:
1. Endpoint `/api/provisao` — já havia sido corrigido antes
2. Endpoint `/api/cashflow` — tinha lógica própria que extrapolava `_cur_ads_actual / elapsed_days * month_days` quando `elapsed_days >= 10`, mas no dia 1 tinha `elapsed_days=1` e não caia no critério — usava o próprio valor parcial

**Fix** em `agent/api.py` (cashflow endpoint, ~linha 2782):
```python
_elapsed_days = now.day
_month_days   = calendar.monthrange(now.year, now.month)[1]
_ads_prorata  = _cur_ads_actual / _elapsed_days * _month_days if _elapsed_days > 0 and _cur_ads_actual > 0 else 0.0

# Buscar mês anterior como fallback
_prev_ads_r   = _ads_conn.execute(...)
_prev_ads_val = float(...)
ads_monthly_last30 = round(_prev_ads_val, 2) if _prev_ads_val > 0 else ads_monthly_total

# Só extrapolar se tiver >= 10 dias de dados
if _ads_prorata > 0 and _elapsed_days >= 10:
    ads_monthly_current = round(_ads_prorata, 2)
else:
    ads_monthly_current = ads_monthly_last30 if ads_monthly_last30 > 0 else ads_monthly_total
```

**Why:** Com menos de 10 dias, extrapolar causa distorção severa (1 dia de ADS × 31 = 31× errado). Usar mês anterior / 30 é estimativa muito mais estável.

**How to apply:** Qualquer projeção de custo mensal baseada em dados parciais do mês corrente deve usar o critério `elapsed_days >= 10` antes de extrapolar. Abaixo de 10 dias, usar o mês anterior como proxy.
