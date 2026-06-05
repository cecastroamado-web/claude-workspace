---
name: nfe-discount-override-descontos-contratuais-por-nf
description: "Tabela nfe_discount_override + helper _compute_customer_discount para overrides pontuais quando a retenção contratual incide sobre base diferente de valor_nota (ex.: HAVAN NF 561 — 2% sobre valor original pré-devolução)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 67d1d038-a21b-4cb3-a143-7040e1b7786b
---

Em mai/2026 foi criada tabela `nfe_discount_override (bling_nfe_id, desconto_valor, note)` no SQLite do ecommerce-agent + helper `_compute_customer_discount(rules, overrides, customer_name, receita, bling_nfe_id)` em `agent/api.py`. Override por NF tem prioridade sobre a regra geral em `customer_discount_rules`.

**Why:** O contrato HAVAN prevê retenção de 2% (1% comercial + 1% logístico) sobre o valor ORIGINAL da nota, mesmo após devoluções parciais. A regra geral aplica 2% sobre `valor_nota` (que é o líquido pós-devolução no banco), o que subestima o desconto quando há devolução. Caso concreto NF 561 (bling_nfe_id 24738765785, XConnect): original R$ 274.400, devoluções R$ 150.250, valor_nota = R$ 124.150, retenção contratual = 2% × 274.400 = R$ 5.488, líquido a receber = R$ 118.662,00.

**How to apply:**
- Endpoints que já usam o override: `order-detail`, `commission-report`, `vendor-roi-report`, `compute_profitability` (cashflow + customer-ranking + profitability + executive).
- Para outras NFs com retenção sobre base diferente: `INSERT INTO nfe_discount_override (bling_nfe_id, desconto_valor, note)`. Não precisa mexer em código.
- `services-profitability` (NFS-e) NÃO usa override — é só para NF-e.
- Quando o caso for sistêmico (toda devolução de cliente X deve usar valor original), considerar refatorar a regra para `valor_nota + valor_devolvido` em vez de override pontual.

**Bugs correlatos corrigidos no mesmo PR:**
- `vendor-roi-report`: query SQL não selecionava `customer_name` → `_roi_desconto_cli` sempre recebia "" e desconto Havan nunca era aplicado. Comissão paga ao vendedor estava sendo calculada sobre lucro inflado em 2% da receita.
- `commission-report`: aplicava o desconto no `lucro_liquido` mas não retornava `desconto_cliente` como campo no JSON. Frontend agora exibe coluna "Desc." na tabela "Vendas com Comissão" (`rentabilidade/index.tsx`).

Relacionado: [[ecommerce-havan-devolucoes]], [[ecommerce-customer-ranking-refactor]].
