---
name: Sheets append_sale_row — fix deslocamento de colunas
description: Bug e fix definitivo da inserção de NF-e no Google Sheets (dados iam para coluna O em vez de A)
type: project
originSessionId: c2f7eae1-140e-4e95-ac10-6c321733ef0e
---
## Bug: `values().append()` deslocava dados para coluna O

**Sintoma**: NF-e HAVAN 000578 e 000579 foram inseridas mas com dados a partir da coluna O (vencimento era a última coluna com dados em toda a planilha, só chegava até linha ~3000), em vez de sempre começar na coluna A.

**Causa raiz**: `spreadsheets().values().append(range="A:O", insertDataOption="INSERT_ROWS")` usa a coluna com MENOS dados como âncora para decidir onde começa a nova linha. Como a coluna O (vencimento) tinha dados só até a linha ~3000, enquanto coluna A tinha até ~3005, o Sheets inseriu em A3001 com dados deslocados para O3001.

**Fix definitivo** em `agent/sheets.py` — função `append_sale_row()`:
```python
# 1. Encontrar última linha usando coluna A como âncora
last_row = self._find_last_row()   # lê coluna A apenas
target   = last_row + 1
gid      = self._get_sheet_gid()

# 2. Abrir linha em branco na posição exata (insertDimension)
self._svc().spreadsheets().batchUpdate(
    spreadsheetId=self._sheet_id,
    body={"requests": [{"insertDimension": {
        "range": {"sheetId": gid, "dimension": "ROWS",
                  "startIndex": target - 1, "endIndex": target},
        "inheritFromBefore": True,
    }}]},
).execute()

# 3. Escrever sempre a partir de A{target} (update, não append)
self._svc().spreadsheets().values().update(
    spreadsheetId=self._sheet_id,
    range=f"A{target}:O{target}",
    valueInputOption="USER_ENTERED",
    body={"values": [row_values]},
).execute()
```

**Why:** `values().append()` não garante âncora na coluna A — depende de qual coluna tem mais dados. `insertDimension` + `update` com range exato é a única forma segura.

**How to apply:** NUNCA usar `values().append()` para inserir linhas em planilhas onde colunas têm comprimentos diferentes. Sempre usar `insertDimension` + `values().update(A{n}:O{n})`.

## Correção dos dados incorretos (mai/2026)
- Deletados registros `synced_nfe` das NF-e 000578 e 000579 do DB
- Reiniciado o serviço → próximo BlingNFSync reinseriu corretamente:
  - NF 000579 → linha 3006 (R$218.300, venc 29/08/2026)
  - NF 000578 → linha 3007 (R$27.000, venc 29/08/2026)
- Usuário deve deletar manualmente as duas linhas deslocadas (dados na coluna O, linhas ~3000-3001)
