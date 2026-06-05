---
name: email-agent — Grupo "Sem Dados" do BigQuery tratado como vazio
description: BigQuery faturamento retorna a string literal "Sem Dados" quando o campo Grupo está em branco. _grupo_is_empty() detecta isso para que grupos_override.json funcione e o expander mostre os clientes corretos.
type: project
originSessionId: cb07f42d-08b2-4671-a1ba-2a9281d1e9aa
---
`load_brasil_billing` aplica `_apply_grupos_override(txs, _load_grupos_override())` antes de devolver os dados. O override só substitui o grupo quando `_grupo_is_empty(t.grupo)` é True.

**Why:** descoberto em 07/mai/2026 durante auditoria do Ranking Brasil. O BigQuery devolve a string literal `"Sem Dados"` (não `None` nem `""`) quando o cadastro está vazio, então a checagem `if not t.grupo` antiga deixava passar 192 de 345 clientes (47,8% da receita) sem nunca aplicar override e sem listá-los no expander para preenchimento manual.

**How to apply:** quando criar nova feature em `app.py` que checa se um cliente tem grupo, sempre usar `_grupo_is_empty(grupo)` em vez de `if not grupo`. A helper trata `{"sem dados","sem grupo","n/a","na","-","—","vazio"}` (case-insensitive) como vazio.

## Como preencher grupos_override.json
1. Abrir página Brasil — Ranking, marcar "Mostrar todos"
2. Expandir "⚠️ N clientes sem grupo cadastrado" (mostra 192 hoje)
3. Para cada CNPJ relevante, editar `agent/financeiro/grupos_override.json` adicionando `"00.000.000/0000-00": "Nome do Grupo"`
4. Override é aplicado por cima do BQ sem alterar a base
