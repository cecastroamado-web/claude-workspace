---
name: ecommerce-havan-em-transito-expansao
description: "ecommerce-agent — coluna/visão \"Em trânsito\" (CD→loja) no ranking de filiais Havan + indicador de expansão de rede (lojas entrando com nossa linha)"
metadata: 
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

## ✅ Painéis de evolução/saúde do canal + alerta de ativação (18/jun/2026)
- **`havan_branch_product` RETÉM histórico diário** (snapshot por `captured_date`, coleta 06h) — começou **05/jun/2026**. Permite séries temporais (≠ "foto de agora"). Hoje 147 lojas (excl CD "CDH BARRA VELHA"); baseline 05/jun = 109. CD identificado por regex `\bCDH?\b|CENTRO DE DISTRIB`.
- **Endpoint `/api/havan/filiais-evolucao`** (api.py, após filiais-report): `serie` (diária: total/ativas/novas estabelecidas-vs-entrando-com-linha), `mensal` (último snapshot/mês), `por_uf`/`por_regiao` (total/vende/venda_30d/novas/parado/ruptura), `kpis` (taxa_ativacao, sell_out_medio/mediana/top, parado/ruptura lojas, distribuicao por faixa 0/1-4/5-14/15-29/30+), `ativacao` (curva entrada→1ª venda). Hook `useHavanFiliaisEvolucao`.
- **Frontend (página Havan):** 2 cards — "Expansão de filiais" (footprint + ComposedChart total/ativas/novas-dia) e "Saúde e geografia do canal" (5 KPIs + distribuição sell-out por faixa + barras por região + tabela por UF + curva). **Penetração** = lojas ÷ total da rede (default **196**, editável via localStorage `havan_rede_total`).
- **Estado atual (18/jun):** penetração ~75%, ativação 70% (103/147 vendem), sell-out média 8,7 / mediana 8 / top 29 un/30d (rede equilibrada), 36 lojas estoque parado, 4 ruptura. Sul concentra (81 lojas, +26 novas); SP/Sudeste fronteira (baixa ativação, recém-chegou).
- **⚠️ Curva de ativação NÃO mensurável ainda:** coleta de 13 dias; das 36 lojas que entraram sem venda, **0 ativaram** (esperando ≤7d). Não há ciclo entrada→1ª venda completo. Calibra com o tempo.
- **Alerta `check-havan-ativacao`** (scheduler.py, diário 09h): lojas que entraram (1ª aparição > baseline), têm estoque, mas seguem sem 1ª venda (`venda_total=0`) há **≥21 dias** (default ajustável `dias=`). Ignora filiais do baseline (entrada real desconhecida); anti-spam `scheduler_state.havan_ativacao_alerted` (sep `|`); `_notify` bool guard. Hoje dispara 0 (mais antiga 7d); 1º disparo possível ~início jul.

# Havan — Em trânsito (CD→loja) + Expansão de rede (16/jun/2026)

Feature implementada a pedido do CFO: saber, por filial, se está **pra receber produto** (estoque saindo
do CD Barra Velha e indo pras lojas) e acompanhar a **expansão** da nossa linha na rede Havan.

## Dado e pipeline
- Portal Havan expõe **`qtdEstoqueTransito`** por filial×produto = trânsito INTERNO Havan (CD→loja, já a
  caminho daquela filial). ≠ do em-trânsito de produto (nossas NFs até o CD).
- `agent/havan_scraper.py`: a matriz `filiais_produto` agora captura `em_transito` (era descartado).
- `agent/db.py`: coluna **`em_transito`** em `havan_branch_product` (migração ALTER + create + insert +
  select de `get_havan_branch_product`). Popula a partir da coleta das 06h (snapshots antigos = 0).
- Frontend `dashboard-ui/src/features/havan/index.tsx`:
  - **Coluna "Em trânsito"** (🚚 qtd) na tabela do Ranking de filiais por produto (só filiais com venda).
  - **Seção "Filiais recebendo (CD→loja)"** — lista TODAS as filiais com `em_transito>0`, **inclusive as
    zeradas de venda**. Exclui o **CD Barra Velha** (`isCD` = regex `\bCDH?\b|CENTRO DE DISTRIB`; é hub,
    não ponto de venda). Badge **🆕** + faixa verde **"Expansão de rede: N lojas"**.

## Definições (importante p/ não confundir)
- **"Nunca venderam"** = `venda_total = 0` → nunca venderam NOSSOS produtos. **NÃO significa filial nova.**
- **Expansão de rede** = loja (não-CD) recebendo (`em_transito>0`) que nunca vendeu nada nosso
  (`venda_total=0`) → loja **existente** entrando com nossa linha pela 1ª vez (footprint novo).
- **Ampliação de mix** = loja que já vende outros itens nossos, recebendo um produto adicional.

## Foto 16/jun/2026 (após coleta manual que rodei p/ popular o em_transito)
- **71 filiais recebendo** (excl. CD) · 168 linhas filial×produto com trânsito no total.
- **18 lojas em EXPANSÃO DE REDE** (venda_total=0, entrando com a linha) — todas cidades **estabelecidas**
  (Porto Alegre, Natal, Sorocaba, Caxias, Ribeirão Preto, São Bernardo, Jundiaí, Mogi, Catanduva,
  Hortolândia, Sta Bárbara, Palmas, Petrolina, Paranaguá, Barra do Garças, Várzea Grande, Chapecó SP,
  Via Expressa Floripa). NÃO são lojas novas — são pontos antigos recebendo nossos produtos pela 1ª vez.
- ~53 lojas ampliando mix (já vendem outros itens nossos).
- O CD Barra Velha aparecia como "nunca vendeu" só porque é o hub (não vende) → excluído.

**Caveat:** a coluna/seção só populam na coleta das 06h; rodei `fetch_havan_report()` + `set_havan_snapshot`
manual em 16/jun (sem disparar alertas) p/ ver no mesmo dia. **Acompanhar a evolução dessas 18 no tempo**
(quantas convertem o trânsito em venda) é o próximo sinal de crescimento a observar.

Relacionado: [[ecommerce-havan-dashboard]], [[ecommerce-havan-estoque-breakdown]],
[[ecommerce-havan-transito-estoque]] (decisão de detecção de recebimento via estoque),
[[ecommerce-havan-liquido-carry-sku]] (margem por SKU).
