---
name: ecommerce-havan-estoque-breakdown
description: "Quebra do estoque Havan (loja/CD/trânsito), Total Havan como base oficial, em-trânsito na reposição"
metadata: 
  node_type: memory
  type: project
  originSessionId: cfd2762d-b075-4e71-8725-e23b190a786c
---

CONCLUÍDO 13/jun/2026. Antes o dash tinha 1 número de estoque só (`qtdEstoqueAtual`) que mistura CD central + loja. CFO viu no portal "estoque atual ~600" vs "ponto de venda ~55" e não batia com o dash (mostrava o do meio, 425).

**Campos da API** `POST /Relatorios/ObterDados` (por filial×produto): `qtdEstoqueAtualLoja` (prateleira/PDV), `qtdEstoqueAtualDeposito` (retaguarda loja), `qtdEstoqueAtual` (= CD+loja+dep, SEM trânsito), `qtdEstoqueTransito` (trânsito INTERNO CD→loja, ≠ em-trânsito do CANAL `em_transito_qtd` = NF nossa não recebida), `qtdEstoquePrevistoTotal` (= atual + trânsito). O CD central aparece como pseudo-filial (CDH BARRA VELHA: estoqueAtual alto, loja 0). Quirk: às vezes loja > atual (impossível, timing deles) → CD derivado clampa em 0.

**Backfill IMPOSSÍVEL:** campos de estoque são foto de AGORA (janela jan=jun); só `qtdRecebimento`/`qtdVendasTotais` variam com a data. Acumula só a partir de jun/2026 nas coletas 06h.

**Base OFICIAL = Total Havan = prateleira+retaguarda+CD+trânsito interno (= previsto, bate com o portal).** Decisão CFO evoluiu: 1º "manter ruptura sobre estoque total"; depois identificou que 425 (atual) estava errado e a base correta inclui o trânsito (~600). Frontend: `totalHavanUn` = loja+dep+cd+transito.

**Implementado:**
- `havan_scraper.py`: soma os 4 campos → `estoque_loja/deposito/cd/transito`; deriva `estoque_cd` (=atual−loja−dep, clamp 0); `cobertura_meses`/`ruptura` recomputados sobre o Total Havan; `cobertura_pdv_meses`/`ruptura_pdv` sobre prateleira (loja+dep). Reposição: campo `em_transito` por filial; `sugestao_repor` desconta o que já vem (max(0, v*2 − estq − trânsito)).
- `db.py`: 4 colunas novas em `havan_product_history` (ALTER + COALESCE 0 p/ linhas antigas).
- Dash (`features/havan/index.tsx`): todos os cards de produto usam `totalHavanUn` + rótulo "Estoque Havan" (KPI Estoque, tabela principal, conciliação, ruptura, parados, velocidade/cob7). KPI "Estoque na Havan" abre quebra prateleira/CD/trânsito; KPI "Ruptura na prateleira" (cobertura só na loja); indicador ⚠ quando fonte inconsistente (loja>atual). Reposição (Pontos/Reposição sugerida/Ruptura na ponta): coluna "Em trânsito" + ordenação agrupada por filial, as que precisam de MAIS unidades primeiro (`reposicaoOrdenada`). "O que mudou" mostra Total Havan (`/api/havan/history` `_total_havan` com fallback ao atual; Δ estoque suprimido na transição quando uma coleta tem quebra e a outra não).
- Per-filial/UF/aging seguem em nível de loja (granularidade diferente — o CD é local único, não é divergência).

Ver [[ecommerce-havan-dashboard]].
