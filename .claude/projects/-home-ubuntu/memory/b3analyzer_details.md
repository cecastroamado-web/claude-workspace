# B3 Analyzer — Detalhes Técnicos

## Arquitetura
```
b3analyzer/
├── cli.py              # 17+ comandos Click
├── config.py           # IBOV_TICKERS, SECTORS, CURATED_PUT_TICKERS, PUTS_CONFIG
├── analysis/           # fundamental.py, technical.py, signals.py, volatility.py, sentiment.py
├── options/            # put_panel.py (v2), black_scholes.py, strategies.py, backtesting.py
├── dashboard/app.py    # Streamlit porta 8501, 6 páginas
├── data/               # brapi_client, yfinance_client, oplab_client, cvm_calendar, models.py
├── display/terminal.py # Rich terminal UI
└── notifications/telegram.py
```

## Scanner de PUTs v2 (Março 2026)
- **35 tickers curados** de setores perenes (exclui varejo, aéreo, construção)
- **5 screening gates**: liquidez, volatilidade, prêmio, risco, fundamentalista
- **Score v2**: 8 componentes (yield 25%, prob 20%, IV 15%, spread 10%, theta 10%, fund 8%, risco 7%, regime 5%)
- **Métricas enriquecidas**: IV/HV ratio, RSI, distância ao suporte, fundamental badge (A/B/C/D), earnings warning
- **PutOpportunity**: 60 campos (dataclass)
- **Dashboard**: KPIs, filtros interativos, bubble chart, gauge IV Rank, IV/HV por ticker, detalhe por ticker com fundamentalista, recomendação AI heurística
- **Semáforo**: VERDE (≥18% + delta 0.15-0.35), AMARELO (≥12% + delta ≤0.45), VERMELHO

## Setores Curados para PUTs
Bancos, Seguros, Energia, Mineração, Siderurgia, Utilities, Papel/Celulose, Consumo/Agro, Indústria, Telecom, Bolsa

## Serviços
- `b3-dashboard.service` (Streamlit porta 8501)
- `b3-alerts.service` (daemon Telegram, intervalo 5min, --with-puts)

## Google Sheets Export (sheets_export.py)
- sheet_id: 1Pck8iO61wMScqC0p9_gfEY8JaBZbZRzBhPU9UTJjHSo
- Token OAuth2: `/home/ubuntu/ecommerce-agent/token_sheets.json`
- **4 abas exportadas**:
  - `Scanner` — todas as PUTs (40 colunas), inclui RSI(2) + Dir.Smart, override semáforo se overbought
  - `Barsi` — PUTs filtradas pelo método Barsi/AGF (29 colunas), indicadores DY projetivo + preço teto
  - `Smart Scan` — scan quantitativo de todos IBOV tickers (23 colunas), RSI(2)/W%R(2)/CumRSI(2)
  - PARAMS/TOP — abas originais do Apps Script (não modificadas)
- Daemon exporta Scanner+Barsi a cada ciclo (5min), Smart Scan a cada 30min

## Smart Evaluate v3 (signals.py) — Mean Reversion Quantitativo
- **Indicadores primários**: RSI(2) ≤ 10, Williams %R(2) ≤ -90, CumRSI(2) < 10
- **Filtro obrigatório**: preço acima SMA200 (bloqueia se abaixo)
- **Confirmações**: MACD turning up, OBV slope >0, Bollinger lower band, SMA50 proximity, IBOV bull
- **Força do sinal**: MUITO FORTE (4+), FORTE (2-3), MODERADO (1)
- **Aprovação**: buy_triggers ≥ 1 AND score ≥ 50 AND above SMA200 AND not IBOV blocked
- **Exit**: RSI(2) ≥ 65
- **Cross-reference com put scanner**: RSI(2) ≥ 90 → put VERDE/AMARELO rebaixado para VERMELHO

## yfinance_client.py
- `_to_sa()` — adiciona .SA exceto para índices (^BVSP etc.)

## CVM Earnings Calendar (cvm_calendar.py)
- Previsão de datas de balanço via CVM Dados Abertos (substitui yfinance earningsDate)
- Pipeline: OpLab CNPJ → CVM cad_cia_aberta.csv → CD_CVM → ITR/DFP filing history
- Algoritmo: média de dias (DT_RECEB - DT_REFER) para mesmo trimestre nos últimos 3 anos
- 81/85 IBOV tickers mapeados com sucesso
- Cache local em `~/.b3analyzer/cvm_cache/` com TTL: 30d cadastral, 7d filings, 30d mapping
- Confidence: HIGH (3+ anos), MEDIUM (2), LOW (1)

## OpLab Enrichment (oplab_client.py: get_stock_enrichment)
- 20+ campos extraídos: iv_current, iv_1y_rank, iv_1y_percentile, iv_6m_rank, ewma_current, garch11_1y, beta_ibov, correl_ibov, short/middle_term_trend, m9_m21, oplab_score, entropy
- IV Rank do OpLab substitui cálculo Black-Scholes (fallback BS mantido)
- Cache TTL "history" (1h)
- Usado em: put_panel (IV Rank direto), signals.py (beta/trend bonuses), sheets_export (5 colunas extras)

## Smart Scan Sheets Columns (28 total)
- Colunas extras: Beta, Trend.CP, Trend.MP, Balanço, Conf.Bal.

## Config tab OpLab referência
- Config tab (gid=1120943620): tickers, parâmetros, semáforo
- Results tab (gid=1264384310): 26 colunas de resultados
