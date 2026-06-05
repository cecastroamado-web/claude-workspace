---
name: DRE Brasil — categorização via categorize_dre (mai/2026)
description: Função categorize_dre em despesas_brasil_reader.py mapeia despesas para categorias DRE com aliases + fallback por pacote_nome
type: project
originSessionId: cb07f42d-08b2-4671-a1ba-2a9281d1e9aa
---
DRE Brasil (`app.py:6125`) usa `categorize_dre()` em `agent/financeiro/despesas_brasil_reader.py` para classificar cada despesa em uma categoria DRE.

**Why:** auditoria 07/mai/2026 mostrou que 3.317 lançamentos (R$ 7,78M acumulado 2024→2026) tinham `sub_centro` vazio mas `pacote_nome` preenchido. A DRE original usava só `sub_centro` direto, então ~50% do OpEx caía em "Outros". Além disso aliases divergiam entre código (`RH`) e planilha (`RECURSOS HUMANOS`).

**How to apply:** ao adicionar nova categoria ou pacote no DRE Brasil, atualizar os dois dicts em `despesas_brasil_reader.py:407-450`:
- `_SUB_CENTRO_ALIASES` — sub_centro da planilha → categoria DRE canonical
- `_PACOTE_TO_DRE` — pacote_nome → categoria DRE (fallback quando sub_centro vazio)

## Categorias DRE canônicas
`MIDIA`, `RH`, `COMERCIAL`, `ADMINISTRATIVO`, `PRODUCAO`, `IMPOSTOS`, `DIRETORIA`, `INVESTIMENTO`, `FINANCEIRO`, `OUTROS`.

## Aliases sub_centro → categoria
- `RECURSOS HUMANOS` → RH (planilha usa esse nome, não "RH")
- `COMERCIAL BR` → COMERCIAL
- `MÍDIA` / `MIDIA` → MIDIA
- `PRODUCAO` / `PRODUÇÃO` → PRODUCAO
- demais nomes batem direto (ADMINISTRATIVO, IMPOSTOS, DIRETORIA, INVESTIMENTO)

## Fallback pacote_nome → categoria (quando sub_centro vazio)
- Custo de Mercadorias Vendidas → MIDIA
- Pessoal - Gastos Diretos/Indiretos → RH
- Impostos sobre Receita / Cobrança, Impostos e Taxas → IMPOSTOS
- CAPEX → INVESTIMENTO
- Diretoria - CEO / DIVIDENDOS → DIRETORIA
- Despesas Financeiras → FINANCEIRO
- Serviços Terceiros / Telecomunicações → PRODUCAO
- Informática / Despesas Gerais / Jurídico / Contratos / Manutenção / Veículos → ADMINISTRATIVO
- Despesas com Vendas / Viagens / Promo → COMERCIAL
- DEVOLUÇÕES CLIENTES → OUTROS (resíduo)

## Linhas DRE finais (em `app.py`)
Receita Gestão → (-) Mídia (CMV) = Margem Bruta → (-) RH/Comercial/Admin/Produção/Impostos/Diretoria/Investimento (CAPEX)/Despesas Financeiras/Outros = Resultado Operacional → (-) Dívidas Parceladas = Resultado Final.

Linha `(-) Atendimento` foi removida (categoria não existe na planilha, era sempre R$ 0).

## Validação 10/mai/2026 — DRE Brasil 2024→2026 parcial
- OUTROS = R$ 9,9k (0,04% do total) — funcionou como esperado.
- Total despesas (sem dívidas parceladas) = R$ 23,85M — confere.
- Resultado Final por ano: -R$ 7,44M (2024) · -R$ 5,61M (2025) · -R$ 3,77M (2026 parc.) — déficit em queda.
- Mídia (CMV) caiu de R$ 4,68M (2024) → R$ 3,10M (2025) → R$ 0,62M (2026 parc.) — pendente investigar se é desativação real, mudança de modelo ou aliasing a corrigir.
- 2 sub_centros não mapeados pendentes (cosmético, R$ 5,8k total): `"COMERCIAL INT"` (R$ 3.976) e `"PASSAGEM AEREA"` (R$ 1.833) — usuário precisa decidir destino antes de mapear.
- App já desvia `is_divida_parcelada` (pacote "Dívidas Parceladas") para linha separada antes de chamar `categorize_dre()` — total R$ 5,72M acumulado em parcelamentos tributários (PERT, Min. Fazenda, ISS municipais).
