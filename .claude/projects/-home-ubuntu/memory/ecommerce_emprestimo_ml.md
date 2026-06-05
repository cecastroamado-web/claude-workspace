---
name: Empréstimo ML XConnect (abr/2026)
description: Empréstimo contratado via ML com retenção de 25% das vendas líquidas diárias XConnect
type: project
originSessionId: 007a3951-8cee-4ab9-911f-651c3ee57166
---
Empréstimo ML contratado em 10/04/2026 para XConnect.

- **Valor contratado (recebido):** R$ 214.000,00
- **Valor a pagar (total):** R$ 246.027,99
- **Retenção:** 25% das vendas líquidas diárias do ML XConnect (retido pelo ML antes de creditar)
- **Início da retenção:** 10/04/2026

**Why:** Capital de giro via antecipação ML, amortizado automaticamente via retenção diária.

**How to apply:** Constantes em `agent/api.py` (`_EMPRESTIMO_*`). A provisão de caixa exibe banner âmbar com retenção diária estimada e data estimada de quitação. Cada dia da tabela mostra linha "Retenção empréstimo ML 25%" como saída. Quando o empréstimo for quitado, zerar `_EMPRESTIMO_VALOR_TOTAL = 0` para desativar.
