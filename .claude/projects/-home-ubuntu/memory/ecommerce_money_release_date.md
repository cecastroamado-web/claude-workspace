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

**Saldo devedor ancorado no valor REAL do ML (11/jun/2026):** a estimativa pela planilha
(246.028 − lançamentos "EMPRESTIMO/MERCADO LIVRE") DEFASA o saldo real do ML (a retenção
25% é na fonte e o lançamento atrasa; ml_orders_sync só tem 60d). Override do CFO:
`scheduler_state` chave `emprestimo_saldo_override` = {saldo, as_of} = valor lido no painel
do ML. Helper `_emp_saldo_real()` (api.py): com override → ancora nele − pagamentos lançados
APÓS `as_of`; sem → estima pela planilha. Usado no cap E no display. `GET/POST
/api/emprestimo-ml-saldo`; banner do ProvisaoCard mostra Pago + Saldo devedor com botão
"atualizar" inline. Valor real 11/jun: pago 104.162 / saldo 141.865 (total 246.028).
O CFO reatualiza quando consulta o ML.

**Cap de quitação (11/jun/2026):** a retenção 25% PARA quando o saldo devedor
(`_EMPRESTIMO_VALOR_TOTAL` 246.028 − pago real da planilha "EMPRESTIMO"+"MERCADO LIVRE")
é coberto. `_emp_ret_cap` lido ANTES dos loops de release, decrementado a cada retenção
(pending, released-hoje, projeção). Antes retinha "para sempre" (sem efeito no horizonte
de 60d pois quita só ~jan/2027, mas era inconsistente com o gráfico que já capava). Paridade
provisão×gráfico confirmada: caixa líquido ML = 0,75×bruto − taxas nos dois (gráfico:
receita_ml_est líq − loan_pgto 25%×bruto; provisão: líq − retenção). Ver [[ecommerce-provisao-grafico-paridade]].

**Frontend:** `dashboard-ui/src/features/cashflow/ProvisaoCard.tsx` reescrito sem antecipação — banner "Liberação ML" mostra "A liberar (real)" = painel, separado da projeção; empréstimo mostra retenção+abatimento. Tipos novos em `lib/api.ts` (`entradas_ml_real/_proj`, `total_releases_reais_periodo`).

**Alerta Telegram (jun/2026):** o job `check-provisao` (`scheduler.py:_run_check_provisao`, diário 08h) **ainda usava o modelo antigo de antecipação** (média diária + "Antecipável 3d @ 4,16%") e ignorava os releases reais — mostrava XConnect despencando p/ saldo negativo no dia 0 (ex.: -5.325) enquanto o dashboard mostrava +668. Reescrito p/ **chamar `get_provisao(empresa=, dias=30)` direto** (mesma função do endpoint; `_verify_token` Depends vira default não-usado quando chamado in-process) e formatar o bloco a partir da resposta: `saldo_atual_total` (banco + "MP a sacar"), `total_releases_reais_periodo` ("A liberar ML real"), projeção futura, médias, bloco empréstimo (se ativo) e a lista `alertas` (threshold amarelo R$ 20k). Fonte única de verdade = endpoint. `_notify` honrado p/ logar envio real.

**Testes:** `tests/test_pending_releases.py` (6 casos: filtro pending, reconciliação Aviation/XConnect vs painel, clamp em 0, preservação do release_date no re-sync).

**Bug status defasado (corrigido 07/jun/2026, `2e95c23` + `d7a65da`):** a Fase 5 só buscava `money_release_date IS NULL` — depois de gravar 'pending' nunca re-consultava, e o status não flipava p/ 'released' quando o MP liberava. Backlog de 99 pedidos (01–07/jun) inflou a entrada de "hoje" da provisão em R$ 18.980 (dupla contagem: dinheiro já liberado/sacado + projetado de novo). Fix: a query também re-consulta `pending` com `date(money_release_date) <= hoje` (flipa na próxima rodada horária); o re-check NÃO usa a janela `data >= 30d` (fulfillment libera ~D+30 e escaparia, virando pending eterno de novo) — a janela só limita o backfill de NULL. Sintoma p/ diagnosticar no futuro: entrada ML "real" de hoje muito maior que o painel "a liberar" do ML. Validação pós-fix: XC 08/06 = 905,30 (painel 905), AV 08/06 = 584,51 (painel 584). Mesmo princípio vale p/ SAÍDAS: contas a pagar de hoje já debitadas antes do snapshot de saldo também duplicam (marcar pagas na planilha A PAGAR).

**Watchdog de release defasado (`08/jun/2026`):** o job `check-provisao` (08h) abre com uma verificação: pedidos `status='paid' AND money_release_status='pending'` com `money_release_date <= hoje−2d` indicam que o re-check horário falhou (API fora, pedido sem payment aprovado, reembolso). Se houver, manda alerta Telegram por empresa (nº de pedidos + líquido travado) avisando que a provisão pode estar inflada. Estado saudável = nenhum defasado (re-check flipa em ~1h). Sintoma que originou tudo: entrada ML "real" de hoje >> painel "a liberar" do ML.

**Mais dois ajustes (07-08/jun/2026):** (1) `46e0fa2` — vencimento em fim de semana rola p/ próximo dia útil (`_next_business_day` em `sheets.py`, aplicado em `get_contas_pagar_por_dia` e `get_a_receber_por_dia`); conta de domingo aparecia debitada no domingo mas só sai segunda. Feriados fora de escopo. (2) `8ee52ff` — o campo `deficit` dos alertas é distância até o piso amarelo de R$ 20k (`_PROVISAO_SALDO_AMARELO`, api.py), NÃO saldo negativo; banner do ProvisaoCard reescrito ("faltam R$ X p/ o piso"). Semáforo: verde ≥50k, amarelo 20–50k, vermelho <20k. Piso é hardcoded e igual p/ as duas empresas (possível melhoria: configurável).

Ver [[ecommerce_provisao_model]], [[ecommerce_icms_model]], [[ecommerce_emprestimo_ml]].
