---
name: ecommerce-agent OAuth tokens
description: Dois tokens Google separados no ecommerce-agent — sheets vs drive — e regras para não quebrar
type: project
originSessionId: 7173cb88-f545-458e-acc4-bda7ae4fff13
---
## Arquitetura de tokens (definitiva — mai/2026)

O ecommerce-agent usa **dois tokens Google separados**:

| Arquivo | Scope | Usado por |
|---|---|---|
| `token_sheets.json` | `spreadsheets` | `agent/sheets.py` — leitura/escrita planilha financeira |
| `token_drive.json` | `drive.readonly` | `agent/api.py` — download do Excel de imãs (Drive ID `1a0G6lDQjMc_B4sIOcVT3iKwZtXKl_mTU`) |

## OAuth client em produção (2026-05-11)

OAuth consent screen do client `245427247669-la5p6dk1fo8fg0vouv50a2n2g3bpl21u` foi promovido para **In production** (status External). Refresh tokens agora deveriam ser permanentes.

**Mas ainda revogam ocasionalmente** — token_drive.json foi revogado em 21/mai/2026 (10 dias após geração). Quando acontece, log mostra `_fetch_imas_data: invalid_grant: Token has been expired or revoked` e o `/api/cashflow` retorna `pipeline_producible=0` e `pipeline_transit=0` silenciosamente.

Causas conhecidas: revogação manual em myaccount.google.com, mudança de senha, conta suspensa, client_secret rotacionado, ou ocasional bug do consent. Investigar brevemente, mas se nada óbvio, regerar e seguir.

## Regras críticas

- **NUNCA adicionar `drive` ou `drive.readonly` ao `token_sheets.json`** — o refresh token desse arquivo só tem autorização para `spreadsheets`. Qualquer outro scope causa `invalid_scope: Bad Request` na renovação → dashboard zera.
- `sheets.py` tem `_BLOCKED_SCOPES = {"drive", "drive.readonly"}` para prevenir regressão no merge de scopes.
- `token_drive.json` tem `drive.readonly` — suficiente para download (não precisamos de escrita no Drive, o usuário edita a planilha manualmente).

## Se precisar re-autenticar

**token_drive.json** — usar `scripts/setup_drive_oauth.py` (flow manual, funciona em servidor remoto):
```bash
python3 scripts/setup_drive_oauth.py
# 1) abre a URL impressa no browser, autoriza
# 2) cola de volta a URL inteira de redirect (http://localhost:8509/?code=...&state=...)
# 3) script extrai o code, troca pelo refresh_token e grava token_drive.json
```

OOB (`urn:ietf:wg:oauth:2.0:oob`) foi descontinuado pelo Google em 2022 — usar redirect_uri `http://localhost:8509` e ignorar o erro de "site não disponível" no browser; o code está na URL.

Para automação via Claude num servidor sem browser, dá pra alimentar stdin via fifo:
```bash
mkfifo /tmp/oauth_fifo
sleep 3600 > /tmp/oauth_fifo &     # holder pra não fechar o fifo
python3 -u scripts/setup_drive_oauth.py < /tmp/oauth_fifo > /tmp/oauth_log 2>&1 &
echo "s" > /tmp/oauth_fifo          # responde "sobrescrever?"
# espera URL aparecer em /tmp/oauth_log, manda pro user
echo "URL_COLADA_PELO_USER" > /tmp/oauth_fifo
```

**token_sheets.json** (Sheets): não tem script equivalente ainda; se precisar, copiar o padrão do `setup_drive_oauth.py` trocando o scope para `spreadsheets` e o arquivo de saída.

**Why:** Histórico de quebras: token_sheets.json foi corrompido com scope `drive` na sessão anterior, causando `invalid_scope` a cada renovação e zerando o dashboard. Solução definitiva: tokens separados com scopes isolados.

**How to apply:** Se o dashboard zerar e os logs mostrarem `invalid_scope`, checar qual token está com scope errado. Nunca misturar scopes entre os dois arquivos.
