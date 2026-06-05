---
name: email-agent dashboard — bugs encontrados na validação Brasil (mai/2026)
description: Bugs encontrados durante validação A1/A2/M3/M4/M2 do dashboard Motorleads em 05/mai/2026 — classificação de plataformas de mídia e vendedor_2 placeholder
type: project
originSessionId: cb07f42d-08b2-4671-a1ba-2a9281d1e9aa
---
Validação dashboard Motorleads — Brasil concluída em 05/mai/2026 (5 itens: A1, A2, M3, M4, M2).

**Why:** revisão sistemática prevista no doc `docs/validacao_dashboard_maio2026.md` (Semana 2 — Altos + Médios).

**How to apply:** ao editar `agent/financeiro/despesas_brasil_reader.py` ou seções de Brasil em `agent/dashboard/app.py`, lembrar dos dois bugs corrigidos abaixo.

## Bug 1 — Classificação de plataformas de mídia (corrigido)
- `_infer_plataforma` em `despesas_brasil_reader.py:407-420`
- Keyword `"ads"` no Google capturava `FACEBOOKADS` e `TIKTOKADS` (dict iterava Google primeiro)
- Impacto histórico: R$ 2,35M de Facebook + R$ 157k de TikTok somados como Google entre jan/25 e abr/26
- Fix aplicado: reordenar dict (Facebook/TikTok/Waze antes de Google) e remover keyword genérica `"ads"`. Manter `"google"` + `"adwords"`

## Bug 2 — vendedor_2 placeholder e duplicado (corrigido)
- `app.py:7048-7051` (página Brasil — Comercial)
- 227 registros mar/2026 tinham `vendedor_2='0'` (string placeholder) → criava vendedor fictício "0"
- 48 registros tinham `vendedor_2 == vendedor` (Carlos Eduardo) → dobrava o próprio crédito
- Fix aplicado: `if v2 and v2 != "0" and v2 != v` antes de creditar
- Resultado: soma de "% Total" caiu de 200% → 103,1% (3,1% acima de 100% pelos 14 casos legítimos)

## Pendência (não corrigida — aguardando confirmação CFO)
- KPI "Margem Bruta" em Brasil — Mídia (`app.py:6975`):
  - Numerador `_total_faturado` = Google + Facebook (faturamento)
  - Denominador `_total_rec` = todas as plataformas (inclui TikTok, Waze, Outros)
  - Margem fica subestimada quando há gasto em plataforma sem faturamento direto
  - Decisão pendente: comparar like-with-like (só Google+FB nos dois lados) ou manter como medida de eficiência total?
