---
name: ecommerce-money-release-date
description: "ecommerce-agent — provisão de caixa usa money_release_date real do ML (modelo sem antecipação desde 20/05/2026), valida contra painel \"a liberar\""
metadata: 
  node_type: memory
  type: project
  originSessionId: e0197d9a-df43-4b31-a9f7-e1b367bd8a70
---

ecommerce-agent: provisão de caixa (`/api/provisao`) migrada para **modelo SEM antecipação** desde 20/05/2026 (último dia que o CFO antecipou). O caixa entra na `money_release_date` REAL do Mercado Pago, não no dia da venda. Concluído 31/05–01/06/2026.

**Escrita (coleta):**
- `db.py`: colunas `money_release_date`/`money_release_status` em `ml_orders_sync`; método `get_pending_releases_real(company, date_from)` (só `status='paid' AND money_release_status='pending' AND release >= corte`).
- `mercadolivre.py`: `fetch_money_release(payment_id)` via `/v1/payments/{id}` do Mercado Pago (`MP_BASE_URL`, mesmo token MLB — o `/orders/{id}` do MLB NÃO traz esses campos).
- `scheduler.py`: Fase 5 do `_run_ml_orders_sync` popula releases; **o `_write_orders` (INSERT OR REPLACE) preserva `money_release_date`/`status` via subconsulta correlacionada** (igual shipping_cost) — antes apagava a cada sync e a Fase 5 repopulava, instabilizando o painel.

**Consumo (provisão, api.py bloco 4):**
- Entrada de caixa bucketizada pela `money_release_date` real (estimado=False).
- **XConnect: entrada já líquida da retenção de 25% do BRUTO** (empréstimo ML retém na fonte). NÃO lança retenção como saída (não chega ao banco); expõe `retencao_emp` por dia + `total_retencao_emp_periodo` como informativo (abatimento do empréstimo).
- Vendas futuras ainda não realizadas: média 14d deslocada pelo lag de liberação por empresa (estimado=True), só para dias onde (D−lag)>hoje (evita double-count com releases reais).
- **Lag de projeção:** usar `db.get_avg_release_lag` (MÉDIA sobre released+pending dos últimos 60d, ~3,7d XC / ~8,7d Av), NÃO a mediana dos pending (enviesada para os lentos fulfillment ~D+30, pois os rápidos já liberaram). Usar só pending dava lag ~11-13 e subestimava a projeção (mês 06: 120k vs 143k corretos). Fix jun/2026.
- Slot rastreia `net_real` vs `net_est`; resumo expõe `total_releases_reais_periodo` (= painel ML) e `total_projecao_futura_periodo`.
- Custo de antecipação 4,16% zerado; campos de antecipação mantidos no schema (zerados).

**Validação vs painel "a liberar" do ML (mai/2026):** XConnect R$ 16.966 (painel 16.975, Δ R$8); Aviation R$ 25.655 (painel 25.954, ~1%). Reconciliação XConnect: net 24.310 − 25%×bruto 29.374 = 16.967. Lag real medido bem menor que o D+14 antigo: XConnect ~3,4d, Aviation ~8,2d (liberação amarrada à entrega + reputação MercadoLíder).

**Frontend:** `dashboard-ui/src/features/cashflow/ProvisaoCard.tsx` reescrito sem antecipação — banner "Liberação ML" mostra "A liberar (real)" = painel, separado da projeção; empréstimo mostra retenção+abatimento. Tipos novos em `lib/api.ts` (`entradas_ml_real/_proj`, `total_releases_reais_periodo`).

**Alerta Telegram (jun/2026):** o job `check-provisao` (`scheduler.py:_run_check_provisao`, diário 08h) **ainda usava o modelo antigo de antecipação** (média diária + "Antecipável 3d @ 4,16%") e ignorava os releases reais — mostrava XConnect despencando p/ saldo negativo no dia 0 (ex.: -5.325) enquanto o dashboard mostrava +668. Reescrito p/ **chamar `get_provisao(empresa=, dias=30)` direto** (mesma função do endpoint; `_verify_token` Depends vira default não-usado quando chamado in-process) e formatar o bloco a partir da resposta: `saldo_atual_total` (banco + "MP a sacar"), `total_releases_reais_periodo` ("A liberar ML real"), projeção futura, médias, bloco empréstimo (se ativo) e a lista `alertas` (threshold amarelo R$ 20k). Fonte única de verdade = endpoint. `_notify` honrado p/ logar envio real.

**Testes:** `tests/test_pending_releases.py` (6 casos: filtro pending, reconciliação Aviation/XConnect vs painel, clamp em 0, preservação do release_date no re-sync).

Ver [[ecommerce_provisao_model]], [[ecommerce_icms_model]], [[ecommerce_emprestimo_ml]].
