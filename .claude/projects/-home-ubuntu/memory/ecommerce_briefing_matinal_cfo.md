---
name: ecommerce-briefing-matinal-cfo
description: Plano (a implementar) de um briefing matinal único no Telegram pós-coleta 06h — o que um CFO precisa saber na 1ª hora
metadata: 
  node_type: memory
  type: project
  originSessionId: 5c7eb85e-fc5e-4fc1-80b9-ec8a21e0ed02
---

PLANEJADO em 11/jun/2026 (a pedido do CFO), **ainda NÃO implementado**. Objetivo: consolidar num
único "Bom dia, CFO" no Telegram, disparado ~06h30 (após a coleta das 06h: sync bancário Inter +
havan-sync + NF-e Bling 06h30), o que importa na 1ª hora. Princípio: **uma mensagem**, de cima
(dinheiro hoje) p/ baixo (tendência); **suprime seção sem sinal** (não polui); cada linha acionável.
Crítico (saldo projetado < mínimo, atraso Havan grande, falha de coleta) também vira push imediato.

Hoje os alertas existem mas ESPALHADOS (saldo 10h, a-vencer 07h45, vendas 09h, ruptura no havan-sync
06h, etc.) — ver [[Alertas CFO ecommerce-agent — jobs do scheduler]]. A proposta NÃO remove os jobs;
agrega num digest matinal. ✅=já existe (reusar fonte)  🆕=novo.

**Seções propostas (ordem = prioridade do CFO):**
1. **CAIXA HOJE** — saldo consolidado/empresa (Inter) + Δ vs ontem ✅; aplicado em RF ✅; a liberar ML
   hoje (provisão) ✅; saldo MP a sacar ✅; títulos a pagar hoje+2d (valor+quem) ✅(check-avencer);
   **posição do dia: entra X − sai Y = sobra/falta Z** 🆕; ⚠️ se saldo projetado < mínimo em algum
   dos próx. 7 dias (fonte /api/cashflow D+90) 🆕.
2. **RECEBÍVEIS/RISCO** — Havan: a receber total + próximas entradas (entrega+120d), em-trânsito (a
   entregar), a faturar (fontes já no /api/havan) ✅; atrasados se houver 🆕; divergência de
   conciliação que peça ação 🆕.
3. **VENDAS (pulso)** — faturamento ontem (ML+Bling+NFS-e) vs média 7d/30d % ✅; mês corrente
   run-rate vs mês anterior/meta 🆕; nº pedidos/ticket ✅. **Projeção de fechamento do mês pela
   velocidade dos últimos 7 dias** 🆕 (pedido do CFO, explícito): mostrar **(a) quanto já vendeu
   no mês (MTD, R$ e qtd)** + **(b) projeção do mês = MTD + velocidade 7d (R$/dia e qtd/dia) ×
   dias restantes**, total e por item; comparar c/ mês anterior e % de evolução. Mesma janela 7d
   do quadro por item.
3b. **QUADRO POR ITEM (D-1)** 🆕 — peça central pedida pelo CFO (campo a campo). UMA linha por
   SKU, com: **vendas do dia anterior (qtd + R$)** · **média 7d × média 30d (qtd/dia) c/ seta ▲▼
   de aceleração/desaceleração** (item esquentando 7d≫30d / esfriando 7d≪30d) · **estoque atual** ·
   **ruptura (status: zerado/negativo/dias de cobertura)** · **pedidos em aberto** · **valor a
   faturar** (= pedidos abertos − em-trânsito × atacado, p/ Havan). Visão UNIFICADA ML + Havan +
   B2B. Ordenar por relevância (giro × R$); destacar o que não vendeu nada vs média e o que está
   em ruptura. Fontes já existentes: `ml_orders_sync` + itens Bling (vendas D-1), médias como a
   sugestão de pedido de imãs ([[ecommerce-ima-sugestao-pedido]]), snapshot Havan
   (pedidos_abertos/em_transito/a-faturar — ver [[ecommerce-havan-backlog-faturamento-transito]]),
   estoque por item. Provavelmente exige consolidar SKUs ML×Havan num catálogo comum.
4. **OPERAÇÃO/RUPTURA** — ruptura ML (parados/sem estoque) ✅; ruptura Havan na ponta ✅; runway de
   imãs + "pedir agora?" considerando lead time ✅(sugestão de pedido); catálogo ML pausado ✅.
   (Parte da ruptura já fica visível no Quadro por item acima; aqui é a visão consolidada/acionável.)
5. **OBRIGAÇÕES/IMPOSTOS** — impostos a recolher na semana (DAS/DARF/ICMS/DIFAL) c/ vencimento 🆕;
   parcelas de empréstimo do mês (Sicredi/ML) 🆕.
6. **SAÚDE DE DADOS** — falha de sync (NF-e integrity ✅, token Drive ✅, portal Havan ✅);
   anomalia de pedidos ✅.

**Decisões pendentes do CFO antes de codar:** (a) digest único vs manter separado + só adicionar os
🆕; (b) quais seções entram no MVP; (c) horário exato (06h30 após NF-e?) e se domingo/feriado roda;
(d) limiar de saldo mínimo p/ o ⚠️ de caixa (hoje SALDO_MINIMO=30k no check-saldo).

Padrão técnico ao implementar: legacy Markdown (sem escapes \), `_notify` retorna bool → só marca
state se True (ver regra em CLAUDE.md/[[ecommerce-alerta-enviado-falso]]); catch-up no restart;
scheduler_state p/ não duplicar. Relacionado: [[ecommerce-provisao-pontos-cegos]],
[[ecommerce-money-release-date]], [[ecommerce-havan-backlog-faturamento-transito]].
