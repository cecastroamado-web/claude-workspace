---
name: ecommerce-overview-categorias-caixa
description: "ecommerce-agent Visão Geral — categorias de caixa, IMPOSTO A RECUPERAR e identidade Variação de Caixa = Δ saldo do banco"
metadata: 
  node_type: memory
  type: project
  originSessionId: b988fa55-62a8-429d-bc3c-83c51c398212
---

# Visão Geral — categorização de caixa e reconciliação

`/api/overview` (`agent/api.py`) + `dashboard-ui/src/features/overview/index.tsx`.

## Identidade que DEVE valer
**Variação de Caixa do período = Δ saldo consolidado no banco.** Saldo do banco
(`/api/balance` → `get_bank_balance`) = soma de TODA linha `STATUS=OK`. A Variação
só fecha com isso se os 4 baldes cobrirem 100% das linhas de caixa:
- operacional = receitas − (despesa_op + pró-labore + juros)
- investimentos = −capex + rf_liquido + **recuperaveis_liquido**
- financiamento = empréstimos recebidos − pagos
- distribuições = −dividendos

Se a identidade quebrar, procurar categorias que entram no saldo mas não em nenhum
balde (ex.: `TRANSFERÊNCIA ENTRE CONTAS` desbalanceada).

## Caso real jun/2026 — diferença de R$ 4.812,63
Variação 62.674 vs banco 57.861. Causa: 2 lançamentos de mar/2026 saídos do INTER
XConnect como `TRANSFERÊNCIA ENTRE CONTAS` SEM perna de entrada (20/03 −3.167,80 e
31/03 −1.644,83). Eram **parcelamento de imposto não acatado, com reembolso pedido**
(dinheiro vai voltar). Transferências entre contas próprias DEVEM zerar — quando não
zeram, é perna faltando ou categoria errada. (As 5×R$150 jul–nov/2025 eram
intercompany XConnect→Aviation e zeram no consolidado.)

## Regra: IMPOSTO A RECUPERAR (commit cef5ee1)
Valor que saiu mas vai voltar (imposto pago em parcelamento negado → reembolso) NÃO
é despesa de P&L nem transferência — é **recuperável** (ativo temporário).
- Planilha: lançamento de saída com DESCRIÇÃO `IMPOSTO A RECUPERAR` (STATUS=OK); e,
  quando souber, um lançamento de entrada `STATUS = A RECEBER` (+valor, sem data) que
  vira OK+data quando o reembolso cai.
- Código: `CATEGORIAS_RECUPERAVEL` (IMPOSTO/IMPOSTOS/VALOR(ES) A RECUPERAR) entra em
  `CATEGORIAS_FINANCIAMENTO` → `_classifica` devolve "financiamento" (fora do P&L em
  todos os consumidores). `get_overview` soma `recuperaveis_liquido` (signed) em
  `fluxo_investimentos` e expõe `recuperaveis_liquido_periodo`. Card Investimentos
  mostra "(−) Impostos a Recuperar (pagos)" / "(+) Reembolso de Impostos".
- Validado: com isso Variação = Saldo banco = R$ 57.781,00 (fecha ao centavo).

Ver [[ecommerce-cashflow-financiado]] (card Renda Fixa, simulações).
