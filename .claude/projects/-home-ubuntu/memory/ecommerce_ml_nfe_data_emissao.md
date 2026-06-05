---
name: ml-nfe-gotchas
description: Dois gotchas estruturais em ml_nfe — data_emissao NULL em ~30% dos registros + linhas duplicadas quando ML consolida pedidos. SEMPRE JOIN com ml_orders_sync e dedupe por invoice_key.
metadata: 
  node_type: memory
  type: project
  originSessionId: 449ba2d4-8551-480d-80ba-72da7b475561
---

## Gotcha 1: data_emissao NULL

A tabela `ml_nfe` (ecommerce-agent, sincronizada do Mercado Livre) deixa **`data_emissao` NULL ou vazia em ~30-50% dos registros**, mesmo quando `invoice_key`, `valor_nota` e `invoice_status='authorized'` estão preenchidos. **Padrão observado em 22/mai/2026**: registros sem `data_emissao` também têm `numero=NULL` — a API ML retorna o registro pré-autorização SEFAZ, com chave e valor mas sem número ainda. Após algumas horas a SEFAZ autoriza e os campos aparecem.

Em abril/2026 XConnect: 352 NFs (32%) tinham `data_emissao` NULL. Em 22/mai o alerta `check-nfe-sync-integrity` disparou com Aviation 41% e XConnect 49,5% — porque o sync tinha **dois bugs** que impediam re-sincronização:

1. **`save_ml_nfe_batch` não atualizava `numero`/`data_emissao` no `ON CONFLICT DO UPDATE`** — só nos valores fiscais. Se a primeira sync veio sem esses campos, ficava NULL pra sempre. Fix em `agent/db.py`: `numero=COALESCE(excluded.numero, numero)` (preserva existente, atualiza só se chegou).
2. **`get_ml_nfe_synced_order_ids` não forçava re-sync** dos registros com data ausente. Só pulava `not_found > 24h`. Fix: também força re-sync de registros com `invoice_key` preenchido mas `data_emissao` ou `numero` NULL após 6h.

## Gotcha 2: NFs duplicadas por consolidação ML

A chave do upsert é `(ml_order_id, company)`. Quando o ML consolida múltiplos pedidos em uma única NF-e (cliente compra 2 itens em pedidos separados), o sync grava **uma linha em ml_nfe por ml_order_id, todas com a MESMA invoice_key**. Resultado: agregações por `SUM(valor_nota)` inflam.

Em mai/2026: 125 invoice_keys distintas estavam duplicadas, totalizando 138 registros excedentes. Para SP April: 3 invoice_keys com 2 cópias inflavam o faturamento em R$ 2.405.

Cleanup feito em 21/mai/2026 (DELETE mantendo MIN(id)). O `agent/db.py:save_ml_nfe_batch` agora detecta consolidação antes do INSERT e PULA quando outro ml_order_id já tem a mesma invoice_key (loga "ml_nfe consolidação ML: ...").

## Como os dois bugs juntos quebraram o painel (mai/2026)

`/api/nfe-filiais` filtrava por `strftime('%Y-%m', data_emissao)` e NÃO dedupava por invoice_key. Resultado:
- SP April real (auditado via XML): 498 NFs / R$ 196.727
- SP April mostrado: 309 NFs / R$ 122.978 (perdia R$ 74k por data_emissao NULL)

Mesma coisa em `/api/sped/efd/sp/{year}/{month}/summary` (painel "Resumo Fiscal Filial SP"). Mesma coisa em RS perdia R$ 50k. Usuário pegou auditando XML × dash.

## Regra: para fechamento mensal, use ml_orders_sync.data + dedupe

```sql
SELECT n.invoice_key, MIN(o.data), MAX(n.valor_nota), ...
FROM ml_nfe n
JOIN ml_orders_sync o ON o.ml_order_id = n.ml_order_id AND o.company = n.company
WHERE n.invoice_key != ''
  AND (n.invoice_status IS NULL OR n.invoice_status NOT IN ('not_found', 'canceled'))
  AND o.status = 'paid'
  AND strftime('%Y-%m', o.data) = ?
GROUP BY n.invoice_key   -- defesa em profundidade contra consolidação ML
```

O painel de rentabilidade sempre usou essa semântica (sem GROUP BY explícito porque não fazia SUM agregado) — por isso batia com o painel ML enquanto o NF-e por Filial e o SPED summary não.

## Guard

Job `check-nfe-sync-integrity` no scheduler (diário 09h45) alerta Telegram quando >5% ou >50 NFs em 30 dias estão com `data_emissao` vazia. Ver [Alertas CFO](ecommerce_alertas_cfo.md). NÃO temos guard pra consolidação ainda — confiar na lógica de prevenção em `save_ml_nfe_batch`.

## Antes de tocar qualquer SELECT em ml_nfe

1. **Fechamento mensal/contábil** → JOIN com ml_orders_sync, use `o.data`, GROUP BY `n.invoice_key`.
2. **Aggregação por NF (SUM/COUNT)** → SEMPRE GROUP BY `n.invoice_key` ou DISTINCT — defesa em profundidade.
3. Sempre exclua `invoice_status IN ('not_found', 'canceled')` (e considere `IS NULL`).
4. Se for sobre a NF-e em si (não a venda) → ok usar `n.data_emissao`, mas `COALESCE(n.data_emissao, o.data)` é mais robusto.
