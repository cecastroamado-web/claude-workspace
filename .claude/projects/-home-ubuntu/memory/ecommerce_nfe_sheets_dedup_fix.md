---
name: ecommerce-nfe-sheets-dedup-fix
description: BlingNFSync barrava NF-e no Sheets por falso positivo da dedup favorecido+valor (clientes B2B com valores repetidos)
metadata: 
  node_type: memory
  type: project
  originSessionId: bab9e9ae-2fdf-443f-9b22-c8e3f06973df
---

Bug (jun/2026): a NF Havan 586 (R$ 47.500, situacao=6) nunca foi lançada no Sheets "A RECEBER" — logo não aparecia no painel Havan/fluxo de caixa como faturada antecipável.

**Causa:** `BlingNFSync._sync_one` (bling_sync.py) tem heurística de dedup por favorecido+valor (±3%) para pegar lançamentos manuais SEM nº de NF. Mas `get_existing_entries_index` (sheets.py) alimentava o set `fav_valor` com TODAS as entradas, inclusive as que já têm nº de NF na col M. A NF Havan 583 (R$ 47.500 idêntico) fez a 586 ser tratada como duplicata e marcada `synced` com `sheet_row_number=0` (pulada). Clientes B2B recorrentes (HAVAN, SOTRIMA) repetem valores → falso positivo garantido. Log: `NF 000586 ignorada — já existe entrada com mesmo favorecido/valor`.

**Fix:** sheets.py `get_existing_entries_index` só adiciona ao `fav_valor` entradas SEM nº de NF (`if not nf`), igual o coverage_map já faz. Entradas com NF já são pegas por número exato via `existing_nfs`. Lint OK.

**Resolução da 586:** removida de `synced_nfe` + sync standalone (`BlingNFSync.sync_all(data_inicio='2026-06-14')`) → lançada na linha 5601 (A RECEBER, venc 14/10/2026 = +120d). Estado: `synced_nfe` row=5601.

**Cuidado operacional:** ao mexer em `synced_nfe` com o serviço rodando há corrida via WAL — o processo antigo restaura o registro. Sempre reiniciar o serviço ANTES de deletar/reprocessar.

**Varredura tipo-586 (só falsos positivos reais, que casaram com NF existente):** além da 586, só SOTRIMA NF 047 e 059. Mas NÃO estão faltando — a receita está no Sheets (lin2187, lin2394) lançada com nº de NF ERRADO (71). Corrigir seria só a col M (71→47, 71→59); não lançar (duplicaria). Ver [[ecommerce-bling-rules]] e [[ecommerce-havan-faturadas-antecipacao]].
