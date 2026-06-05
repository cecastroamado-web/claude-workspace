---
name: Brasil — pipeline de projeção C revisado (mai/2026)
description: build_fluxo_brasil_proj usa Vcto + delay calculado do BQ + replicação forward, análogo ao Internacional
type: project
originSessionId: c738a5e8-9708-47a0-9874-623b8cc12be8
---
Implementado em 10/mai/2026 o pipeline "C revisado" para projeção do fluxo Brasil — análogo ao `build_fluxo_internacional_proj`. Disponível na página Fluxo de Caixa Diário como 3º modo: "Avançado (BQ delay + proj. forward)".

**Why:** validação prévia mostrou que delay médio Brasil é -0,7d (mediana 0d) — clientes pagam no Vcto. Não vale cadastrar `payment_terms` para Brasil (custo de manutenção alto, ganho marginal). O ganho real é a **replicação forward** dos clientes recorrentes para meses M+1, M+2 quando ainda não foram faturados — que o Internacional já tem mas o Brasil não tinha.

**Funções novas em `agent/financeiro/cashflow_daily_reader.py`:**
- `compute_brasil_delay_profile(token_path)` — calcula `avg_delay`, `mediana`, `std` por CPF_CNPJ a partir do BQ (Vcto vs Dt_Ultimo_Pagamento). Filtra com `min_pagamentos=5` e retorna apenas clientes "atípicos" (avg > 5d ou std > 7d). Validação real: 32 clientes c/ perfil atípico de ~621 elegíveis.
- `fetch_entradas_brasil_proj(horizon_days, *, project_future_months, delay_profile)` — Vcto como expected default; aplica delay quando cliente está no profile; replica forward para M+1, M+2 dentro do horizonte (deduplica por cnpj×mês).
- `build_fluxo_brasil_proj(saldo, horizon, *, use_historico=True, delay_profile=None)` — monta fluxo completo (entradas proj + saidas histórico OK + comps + dívidas). Se `delay_profile=None`, calcula automaticamente.

**Validação 60d (saldo=0):**
- `build_fluxo_brasil` (atual): R$ 570k entradas, R$ 1,2M saídas, líquido -R$ 631k
- `build_fluxo_brasil_proj` (novo): R$ 1,058k entradas (+85%), R$ 772k saídas, líquido +R$ 285k
- Composição entradas: 54% reais (Brasil) + 46% projetados forward.

**How to apply:** ao tocar projeção Brasil, manter padrão "Vcto + delay só para atípicos + replicação forward". Override manual via `payment_terms.json` Brasil é desnecessário pra varejo (mediana 0d). Reservar para casos contratuais especiais.

**Pendência (não bloqueia):** Brasil ainda não entra em `CashFlowProjection` (Projeção, Cenários Runway, P&L por País continuam só Internacional). Para isso seria preciso adapter `BillingTransaction → InvoiceRecord` e refatorar `build_cash_flow_projection`. Esse próximo passo só vale a pena se houver demanda concreta.

**Limites do delay_profile:**
- Limiares: `_BRASIL_DELAY_MIN_AVG=5.0`, `_BRASIL_DELAY_MIN_STD=7.0`, `_BRASIL_DELAY_MIN_PAGTOS=5`. Ajustar se a base mudar.
- Top atípicos: IDEALE (+11,9d), Arace Casa de Vivencia (-11,0d), Ditrasa (-10,3d), iCarros (+5,8d std 18,6 — alta variância).
