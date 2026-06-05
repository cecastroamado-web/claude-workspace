---
name: ecommerce-agent — sale_fee ML é por unidade, não total
description: ML API order_items[].sale_fee é por unidade — sempre multiplicar por quantity ao somar comissão de pedido
type: project
originSessionId: 5bafddaf-39d3-471e-a358-28ba5e99d728
---
API do Mercado Livre: `order_items[].sale_fee` é a **comissão por unidade**, não por linha de item. Para obter a comissão total do pedido, sempre multiplicar por `quantity`.

**Why:** Bug descoberto em mai/2026 no pedido CONTREX 2000016260293060 (3 unidades, R$1.157,28). O painel ML mostrava comissão de R$114,66 (9,91%), mas o banco gravava R$38,22 (3,30%) — a comissão por 1 unidade. 499 pedidos qty>1 estavam com lucro inflado em ~R$76 cada (~R$39k de comissão fantasma no agregado).

Curiosidade: o "estorno" que aparece no extrato ML (ex: R$35,79) é desconto/bonificação que já está embutido no `sale_fee` por unidade — não é tarifa adicional. Conta confere: tarifa bruta 150,45 − estorno 35,79 = 114,66 = sale_fee_unit × qty.

**How to apply:** Em qualquer leitura de `order["order_items"]` da API ML para calcular comissão, usar `sale_fee * quantity`. Pontos no código:
- `agent/mercadolivre.py:get_order_fees` (corrigido mai/2026)
- `agent/api.py:sync-ml-orders` (corrigido mai/2026)
- `agent/api.py:/api/order-items` display (corrigido mai/2026)
- `agent/scheduler.py:584` no sync periódico de ML orders (corrigido 15/mai/2026 — passou batido no fix original e sobrescrevia o backfill a cada sincronização). Backfill de 184 pedidos rodado em 15/mai (R$ 13.031,40 de fee subestimada recuperada).

**Validação adicional 15/mai:** o "abatimento" que aparece na UI do ML (ex: tarifa 145,86 com abatimento de 67,32) **não é estorno** — é o desconto promocional já embutido no `sale_fee` por unidade que a API retorna. Não precisa subtrair nada; basta multiplicar por qty. Mesmo princípio se aplica ao frete: `shipping.cost` da API já vem com a promoção de 50% aplicada.

**Bug bônus corrigido junto:** `ON CONFLICT(ml_order_id, company)` em `sync-ml-orders` falhava porque `ml_orders_sync` tem PK só em `ml_order_id`. Trocado para `ON CONFLICT(ml_order_id)`.

Pedidos com qty=1 não foram afetados (sale_fee × 1 = sale_fee).
