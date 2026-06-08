---
name: ecommerce-provisao-release-staleness
description: Provisão ecommerce — saldo MP zerado (saque diário) + fix da data de liberação preliminar congelada que criava buraco falso no fluxo
metadata: 
  node_type: memory
  type: project
  originSessionId: a664a5c8-050c-42ae-a666-0f19a2fab470
---

Dois bugs do `/api/provisao` (fluxo de caixa) corrigidos em **08/jun/2026**, descobertos quando o saldo projetado não batia com banco+entradas−saídas.

**1. `saldo_mp_disponivel` era lixo e inflava o caixa de partida.**
- A projeção partia de `saldo_inicial(banco) + saldo_mp_disponivel`. O `saldo_mp_disponivel` vinha de `fetch_mp_balance` (`mercadolivre.py:1034`), que lia o `BALANCE_AMOUNT` da última linha-DADO do Release Report. **Errado:** o CSV é segmentado por `BUSINESS_UNIT`/`SUB_UNIT` (a última linha calhava de ser o sub-ledger de disputas, perto de zero), e a API direta de saldo (`/v1/account/balance`, `/mercadopago_account/balance`) dá 404/403 forbidden p/ este perfil. Valor gravado XC 2.406 / AV 1.855 vs real XC 964 / AV 605.
- **Decisão de negócio (CFO):** o saldo do ML é **sacado todo fim de dia pro Inter**. Logo o MP nunca é caixa de partida separado: saldo de dias anteriores já virou banco (está no `saldo_inicial`), releases de hoje viram banco hoje à noite (já entram como `entradas_ml` do dia). Somar = dupla contagem.
- **Fix:** `saldo_mp_disponivel = 0.0` fixo em `api.py` §4.2 (mantido no schema p/ compat do frontend). Removida a leitura/upsert de `fetch_mp_balance` no job `payouts-sync` (`scheduler.py`). Saque (payouts) continua sendo lido. Trade-off aceito pelo CFO: zerado é conservador (ignora ~R$1,5k que está no MP e cai amanhã), mas sem número errôneo.

**2. Data de liberação preliminar congelada → buraco falso de ~10 dias.**
- O MP dá uma `money_release_date` **preliminar lá na frente (~D+28)** logo após a venda e vai **antecipando** conforme o pedido é entregue (acaba liberando ~D+1 a D+3). A Fase 5 do sync (`scheduler.py:719`) só re-consultava pending cuja data **já passou** (`date(money_release_date) <= date('now')`). Datas preliminares estão no futuro → nunca re-consultadas → congelavam. Resultado: provisão via 0 entrada por ~10 dias enquanto o painel "a liberar" do ML mostrava R$12k+ entrando (XConnect). Validado: re-consultando 75 pending, 55 mudaram de data e passaram a bater 1:1 com o painel.
- **Fix:** re-consultar **TODOS** os pending (`OR money_release_status = 'pending'`, sem filtro de data). Conjunto pequeno (dezenas) e auto-drenante. + backfill one-time re-consultando todos os pending das 2 empresas.
- Cuidado: `money_release_status` é NULL p/ ~10k pedidos antigos (Fase 5 só busca venda dos últimos 30d sem data) — ok, são antigos já liberados/sacados, não afetam projeção.

Relacionado: [[ecommerce-money-release-date]], [[ecommerce-provisao-pontos-cegos]], [[ecommerce-mp-payouts]].
