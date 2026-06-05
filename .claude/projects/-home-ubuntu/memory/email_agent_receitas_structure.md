---
name: email-agent Receitas tab data structure
description: Estrutura completa da aba Receitas Google Sheets — colunas, fluxo de dados, separação serviço vs mídia, e como cada campo é usado na projeção de fluxo de caixa
type: project
originSessionId: 621b5cc7-1075-45f1-bd33-104fa36590ef
---
## Aba Receitas — Colunas (_COL_* em agent/payments/utils.py)

| Constante       | Índice | Conteúdo |
|----------------|--------|----------|
| `_COL_CLIENT`   | 1      | Nome do cliente |
| `_COL_STATUS_FAT` | 10   | Status: "A FATURAR", "FATURADO", "NEUTRA", "CANCELADA", "JURIDICO" |
| `_COL_MES_REF`  | 21     | Mês de referência da competência (int) |
| `_COL_ANO_REF`  | 22     | Ano de referência da competência (int) |
| `_COL_ADWORDS`  | 24     | Google Ads — MÍDIA pass-through |
| `_COL_FACEBOOK` | 25     | Facebook Ads — MÍDIA pass-through |
| `_COL_TIKTOK`   | 26     | TikTok Ads — MÍDIA pass-through |
| `_COL_MAPS`     | 27     | Google Maps — MÍDIA pass-through |
| `_COL_SMM`      | 28     | Social Media Management — SERVIÇO |
| `_COL_SETUP`    | 29     | Setup — SERVIÇO |
| `_COL_FEE`      | 30     | Fee de gestão — SERVIÇO |
| `_COL_TOTAL`    | 31     | Total bruto sem IVA (fee + mídia) |
| `_COL_TOTAL_IVA`| 32     | Total com IVA |
| `_COL_DT_COB`   | 35     | Data de Cobrança (due date na Receitas) |
| `_COL_DT_PAG`   | 36     | Data de Pagamento (preenchida = pago) |

**Separação serviço vs mídia:**
- Mídia = ADWORDS + FACEBOOK + TIKTOK + MAPS (pass-through ao cliente)
- Serviço = FEE + SMM + SETUP (receita operacional da agência)
- Total = Mídia + Serviço (= col 31)

## Fluxo de dados: Receitas → media_monthly → projeção

### `_read_payment_history_impl` (reader.py:305) produz:
1. `history: dict[str, list[int]]` — `cliente_lower → [delay_dias]` (DT_PAG - DT_COB)
2. `country_delays` — delays por país
3. `media_monthly: dict` — ver estrutura abaixo
4. `client_pay_dates` — datas de pagamento históricas por cliente
5. `paid_periods: set[tuple(client_lower, year, month)]` — competências já pagas

### `media_monthly` — estrutura de chaves:

**Chave `(country, year, month)`:**
```python
{
    "adwords": float, "facebook": float, "tiktok": float,
    "maps": float, "smm": float, "setup": float, "fee": float,
    "total": float, "total_iva": float, "currency": str
}
```

**Chave `"__client_media_pct__"`:**
```python
{ "cliente_lower": ratio_midia_sobre_total }  # últimos 12 meses
```
Cálculo: `(adwords + facebook + tiktok + maps) / total`

**Chave `"__client_detail__"`:**
```python
{ (client_lower, country, year, month): {
    "client": str, "country": str, "currency": str,
    "fee": float,    # fee + smm + setup
    "media": float,  # adwords + facebook + tiktok + maps
    "total": float   # total col 31
}}
```

### `get_pending_invoicing` (reader.py:502) retorna:
- Status filtrado: apenas `"A FATURAR"`
- Período: mês atual + mês anterior
- Campos: `country, flag, currency, client, month, year, total, total_iva`
- **Limitação**: NÃO retorna colunas individuais (fee, adwords, etc.) — só total
- Para incluir "A FATURAR" na projeção com split serviço/mídia correto, precisa ler colunas individuais da aba Receitas e passar `fee` e `media` separados

### `build_cash_flow_projection` (projection.py:14) usa:
- `media_monthly["__client_media_pct__"]` para split de cada fatura em aberto via fuzzy match
- `media_monthly[(country, y, m)]` para calcular médias históricas (avg_adw, avg_fb, avg_fee etc.)
- `media_monthly["__client_detail__"]` para detalhe de receita por cliente (revenue_client_detail)
- Função `_get_media_pct(client)` com fuzzy match: busca exato → substring → 0.0

## Como incluir "A FATURAR" corretamente na projeção

**Why:** `get_pending_invoicing` retorna apenas `total`, não a decomposição fee/mídia.
Se usar `_get_media_pct(client)` para dividir o `total`, funciona apenas se o cliente tiver histórico.
Se a função retornar 0.0 (sem match), tudo vai para serviço — inflando `recv_service_month`.

**Solução correta:** Estender `get_pending_invoicing` para retornar `fee` e `media` separados
lendo as colunas individuais (col 24-30), e passar esses valores diretamente para a projeção
sem depender do `_get_media_pct`.

```python
# No dict retornado por get_pending_invoicing, adicionar:
"fee": fee + smm + setup,   # col 28+29+30
"media": adw + fb + tt + mp  # col 24+25+26+27
# Assim a projeção usa valores reais, não ratio estimado
```

## Regra crítica de status na Receitas:
- `"FATURADO"` → já aparece na aba de cobrança (a_vencer/vencido) → NÃO duplicar na projeção
- `"A FATURAR"` → não está na aba de cobrança → pode ser adicionado à projeção
- `"NEUTRA"`, `"CANCELADA"`, `"CANCELADO"`, `"JURIDICO"` → ignorar (skip em _SKIP_STATUS)
