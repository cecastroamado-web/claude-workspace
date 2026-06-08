---
name: ecommerce-ci-github
description: CI do ecommerce-agent (ruff+mypy+pytest no push) — armadilhas que já quebraram o pipeline
metadata: 
  node_type: memory
  type: project
  originSessionId: cdca63bc-a576-479d-afdf-799ffd73b806
---

O repo `cecastroamado-web/ecommerce-agent` tem CI (`.github/workflows/ci.yml`) que roda `ruff check .` → `mypy agent/` → `pytest tests/` em todo push no master. Quebrou silenciosamente por dias em jun/2026 (só e-mails do GitHub avisavam). Corrigido em 07/jun/2026 (commits `972fb4b` + `f9f8c98`).

**Why:** o working tree local sempre "funciona" porque a produção tem o banco e o ambiente completos — o CI roda num ambiente limpo e expõe diferenças.

**How to apply:**
- Coluna nova no SQLite de produção via ALTER manual → SEMPRE adicionar também à lista de migrações idempotentes em `agent/db.py` (init), senão bancos novos (CI/testes) quebram com 500.
- `agent/api.py` tem side effect de import (`_migrate_db()`), guardado por `if DB_PATH.exists()` — não remover o guard; novos side effects de import devem seguir o mesmo padrão.
- A coleta do pytest importa TODOS os módulos de teste antes dos fixtures — `tests/test_pending_releases.py` importa `agent.api` no topo, congelando constantes de env. O fixture de `tests/test_api.py` força credenciais via `patch.object`; testes novos de API devem usar esses fixtures.
- Refactors de contrato (ex.: `dc8cca6`: `_create_with_retry`, `comissao_ml` por `_source`, CMV sem estimate) precisam atualizar os testes no MESMO commit — rodar `pytest tests/` completo, não só o módulo tocado.
- Atenção: pode haver outra sessão Claude commitando no mesmo working tree em paralelo — edits não commitados podem ser absorvidos pelos commits dela (aconteceu em 07/jun). Conferir `git log` antes de commitar.
