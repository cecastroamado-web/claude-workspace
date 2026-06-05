---
name: Resultado por País — revisão pendente
description: Revisar com calma a página "Resultado por País" do dashboard Internacional (email-agent) — base mista accrual+cash, semântica das colunas Dívidas e IVA, e separação vs Fluxo de Caixa.
type: project
originSessionId: 1b7f0fb2-381d-47b9-b9ac-d04445def1f0
---
# Pendente — Revisar página "Resultado por País" (email-agent dashboard)

Local: `agent/dashboard/app.py:1387+` (page == "Resultado por País").

**Status atual (2026-05-04)**: lógica mantida, ajustes aplicados:
- Divisor uniforme `n_period` para todos os países (era `n_ac` por país, distorcia médias quando país só tinha dado em parte do período).
- 1 mês = mantém mês atual com aviso de parcial; 2+ meses exclui o atual.
- IVA usa fórmula competência `max(0, total_iva − total_faturado)` da aba Receitas (alinhada com DRE/P&L/Comparativo MoM).
- Adicionada coluna `Dívidas Prev.` (parcelas mensais ativas da aba Dívidas) ao lado de `Dívidas Pagas` (realizado banco).
- Bloco "📋 Parcelas de Dívida Previstas por País" com expander e resumo USD.
- Bloco "🔮 Cruzamento Realizado × Projeção (mês corrente)" — Receita/OpEx/Dívida Real vs Prev por país via `CashFlowProjection.receivables_detail_all` + `read_commitments` + `read_debt_installments`.

**Decisão conceitual (2026-05-04, validada com usuário):**
- **DRE (Competência)** usa `read_debt_installments` → parcelas contratuais (o que deveria sair).
- **Resultado por País** usa `by_pacote` realizado do banco → o que efetivamente saiu.
- A diferença entre as duas é sinal de atraso/antecipação de dívida — informação útil, manter os dois modelos.

**Pontos a revisar com calma em outra sessão:**

2. **Página mistura duas visões**: tabela superior é Resultado Operacional (accrual) + tabela inferior é Fluxo de Caixa (cash). Decisão pendente: refator B (tabs separados) vs C (eliminar duplicação com Fluxo de Caixa dedicado).

3. ~~**IVA pago vs cobrado**~~ — **RESOLVIDO 10/mai/2026**: tabela agora tem 2 colunas separadas (IVA Cobrado por competência da aba Receitas, IVA Pago do banco). Diferença entre elas sinaliza recolhimento adiantado/atrasado.

4. ~~**Result. Caixa quando há atraso de dívida**~~ — **MITIGADO 10/mai/2026**: adicionada coluna "Atraso Dívida" = Dívidas Prev. − Dívidas Pagas (positivo em vermelho = pagou menos do que devia). Caption também alerta.

**Correções aplicadas 10/mai/2026:**
- **Brasil excluído consistentemente** do bloco accrual e do cruzamento (antes aparecia só no Resultado, faltava no Cruzamento). Brasil tem páginas próprias.
- **`_ref_y, _ref_m` fix**: agora usa `period_eff[-1]` (não mais último mês da janela total, que podia ser o atual excluído).
- **Saldo Atual movido para bloco próprio** abaixo da tabela (era misturado com médias mensais — unidades diferentes).
- **IVA negativo**: removido `max(0, total_iva - total_faturado)` — negativo aparece se houve nota a menor que IVA.
- **Atraso Dívida**: nova coluna com semáforo invertido (positivo=vermelho).
- **IVA Cobrado vs IVA Pago**: 2 colunas separadas.
- Caption do título: explícito que é só Internacional (6 países LATAM), Brasil tem páginas dedicadas.

**Why:** página é referência rápida para operação, mas mistura conceitos que em outras páginas são separados — vale alinhar quando houver tempo de pensar bem nas trade-offs com o negócio.

**How to apply:** quando retomar, ler primeiro `docs/criterios_financeiros_internacional.md` e a memória `email_fluxo_diario_intl.md` (estrutura padrão DRE Internacional). Confirmar com usuário a decisão de #2 antes de mexer — define o resto.
