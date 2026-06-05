---
name: ecommerce-b2b-inventory
description: "Planilha Google Sheets \"Estoque B2B XConnect\" — produtos acabados vendidos por vendedor interno (não ML). Criada em mai/2026."
metadata: 
  node_type: memory
  type: reference
  originSessionId: 9b194c56-682c-4628-ae1d-b8869c9ecd22
---

# Estoque B2B XConnect — Planilha Google Sheets

**ID:** `1XpSzBckN8WLO1abUrTD1syELCOHQmkL1OlRLZi__jpY`
**URL:** https://docs.google.com/spreadsheets/d/1XpSzBckN8WLO1abUrTD1syELCOHQmkL1OlRLZi__jpY/edit
**Proprietário:** `xconnectimport@gmail.com` (conforme padrão [[feedback-ecommerce-google-assets]]). Foi criada erroneamente em `cecastroamado@gmail.com` em 2026-05-13 e a propriedade foi transferida em 2026-05-15 após o backend quebrar com 404.
**Criada em:** 2026-05-13

## Propósito
Estoque de produtos acabados XConnect vendidos por **vendedor interno** (B2B), **não anunciados no Mercado Livre**. Separada da planilha de imãs ([[ecommerce_cases_capacidade]]) porque imãs são matéria-prima de produção de cases e a planilha deles tem fórmulas/abas próprias — misturar quebraria as fórmulas existentes.

## Estrutura (3 abas)
- **Parametros**: SKU | Descrição | NCM | Custo Médio | Estoque Mínimo | Observação
- **Entradas 2026**: Data | SKU | Qtd | NF nº | Custo Unit. Líquido | Valor Total | Fornecedor | Obs
- **Saídas 2026**: Data | SKU | Qtd | NF nº | Valor Unit. Venda | Valor Total | Cliente | Vendedor/Obs

Fluxo: usuário lança **só produto + quantidade + data** nas abas Entradas/Saídas; reader computa saldo = soma(entradas) − soma(saídas) por SKU.

## Produtos cadastrados inicialmente (3 NFs DIGITALLI)
- **ESTACAO-DC-80KW-CCS2** (Estação DC 80kW 2 CCS2 MOD. C — cód. fornecedor 5000.043, NCM 8504.40.10) — 1 un, custo R$ 32.088,50 / venda R$ 88.486,55 / **margem 63,74%** (NF 4169, 18/02/2026)
- **ESTACAO-DC-40KW-CCS2** (Estação DC 40kW 1 CCS2 MOD. C — cód. fornecedor 5000.045, NCM 8504.40.10) — 2 un, custo médio R$ 21.913,57 / venda R$ 56.553,00 / **margem 61,25%** (NFs 4232 16/03/2026 e 4388 30/04/2026)
- **Totais:** estoque a custo R$ 75.915,64 / a venda R$ 201.592,55 / margem bruta consolidada 62,34%

## Snapshot retroativo aplicado
Tabela `estoque_snapshots` em `data/ecommerce.db`: adicionado `tipo='b2b'`, `empresa='XConnect'` para datas 2026-03-31 (2 un, R$ 55.224,06) e 2026-04-30 (3 un, R$ 75.915,63). Sem alterar snapshots ML/imãs.

## Status — CONCLUÍDO (verificado jun/2026)
Toda a integração já está no código:
- Reader `_fetch_b2b_inventory_raw()` — `agent/api.py:8408`
- Endpoint `GET /api/b2b-inventory` — `agent/api.py:8542`
- Seção "Estoque B2B XConnect" no front — `dashboard-ui/src/features/estoque/index.tsx:1060` (+ hook `useB2bInventory`, `api.ts`, snapshot `XConnect_b2b`)
- Config `B2B_INVENTORY_SHEET_ID` — `agent/config.py:64`

## Regras de custo (NF de compra → custo líquido)
Para XConnect (Lucro Presumido desde 2026), aplicado às NFs DIGITALLI (CFOP 5101 intra-RS, CST 051):
- **ICMS destacado** (17%) = **crédito** (subtrai do custo)
- **IPI** (5%) = **custo** (não credita — XConnect é comercial, não industrial)
- Fórmula: `custo_líquido = valor_produtos + IPI − ICMS`

## Fluxo de leitura de NF de entrada via Telegram (mai/2026)
Bot do ecommerce-agent (`agent/telegram_client.py`) aceita PDF com caption `NF ENTRADA` **OU** responde "NF ENTRADA" depois do PDF — salva em `/home/ubuntu/nfs_entrada/<timestamp>_<filename>.pdf` + `.txt` extraído via docling. Também aceita `cancelar` pra descartar PDF pendente.
