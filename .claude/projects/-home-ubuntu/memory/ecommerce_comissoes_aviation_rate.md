---
name: Aviation Simples 11% fixo na apuração de comissões (decisão de negócio)
description: commission-report e vendor-roi-report usam 11% fixo para Aviation, NÃO a alíquota real do simples_rate_history — não é bug
type: project
originSessionId: c55d1021-f78f-437e-a9da-511f2b2ea0d7
---
Apuração de comissões Aviation usa **11% fixo** como alíquota Simples Nacional (do `company_tax_config.simples_rate`), e **não** a alíquota real variável por mês do `simples_rate_history` (4-9% conforme RBT12).

**Why:** Decisão de negócio do usuário (mai/2026). A apuração de comissão precisa de uma base **estável e previsível** para os vendedores — não pode oscilar conforme o RBT12 cresce. A diferença vai pra empresa (lucro maior), não pra comissão variável.

**How to apply:** Não tentar "consertar" essa divergência entre `commission-report`/`vendor-roi-report` e `compute_profitability` para Aviation. As 3 fontes vão divergir nessa parte e está certo. Validações Aviation: imposto = receita × 11%. NF Aviation jan/26 R$ 57.700 → imposto comissão R$ 6.347 (11%); imposto rentabilidade real R$ 4.097 (7,10%). Os dois estão certos no contexto deles.

**Aplicado a:** Aviation apenas. XConnect (Lucro Presumido) usa o cálculo padrão (PIS+COFINS+IRPJ+CSLL+ICMS líquido) consistente entre os endpoints.
