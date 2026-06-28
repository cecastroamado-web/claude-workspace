---
name: ecommerce-receita-sem-nota
description: ecommerce-agent — auditoria de RECEITA SEM DOCUMENTO FISCAL (vendas recebidas sem NF-e) p/ o DRE de gestão; revisão manual em curso
metadata:
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Receita sem documento fiscal (vendas sem NF-e) — auditoria p/ DRE (CFO 15/jun/2026)

## 🟡 ESTADO 27/jun/2026 (retomada — em andamento, NADA gravado ainda)
**Decisão de design do CFO p/ o DRE:** lançar como **LINHAS SEPARADAS** — uma "Receita sem documento
fiscal" **por cliente** + uma **linha de CMV correspondente** (CMV por ano: 2024 31% / 2025 43% /
2026 38%), sem imposto/comissão (margem pura). Isso **VAI MEXER no resultado do DRE de hoje** (sobe
receita e sobe CMV).

**Achado crítico:** só a **RONDOLINK** está de fato lançada (`customer_semnota_adjust` + Rentabilidade).
Os **7 "confirmados grandes" foram DECIDIDOS em 15/jun mas NUNCA persistidos** — não estão na
Rentabilidade nem no DRE. Precisam ser gravados.

**Quebra por ANO dos 7 confirmados — computada 27/jun do Sheets (recebido − NF sit.6, fator 1):**
| Cliente | 2024 | 2025 | 2026 | Total |
|---|--:|--:|--:|--:|
| Israel Teixeira | 70.200 | 119.700 | 6.200 | 196.100 |
| Startecno | 117.500 | — | — | 117.500 |
| Star2go | — | 42.635 | 83.950 | 126.585 |
| Thiago Velloso | 10.221 | 51.800 | 12.810 | 74.831 |
| Welliton Mario | 17.625 | 33.975 | — | 51.600 |
| Juliano Pich | — | 40.480 | — | 40.480 |
| Augusto Franca | — | 33.400 | — | 33.400 |
| **TOTAL** | **215.546** | **321.990** | **102.960** | **640.496** |

⚠️ **2 pontos a confirmar c/ CFO antes de gravar** (valores subiram vs memória 15/jun):
1. **Israel 196.100** (era ~159.100): inclui Pix `"Cp:18236120-Isra…"` R$105.000 + "Israel Teixeira Neto
   Nascimento" R$91.100 — confirmar se o Pix de 105k é dele.
2. **Star2go 126.585** com R$83.950 em 2026: CFO disse que faturou a partir de 2026, mas a busca de NF
   sob "STAR2GO" deu 0 (pode estar com outro customer_name) → se há NF 2026, o excedente real é MENOR.
Os outros 5 batem. Script da apuração: `/tmp/.../scratchpad/semnota_7.py` (lê Sheets 1×, fator 1, mostra favorecidos casados).

**Lista a validar (CFO faz AMANHÃ 28/jun):** XLSX `receita_sem_nota_REVISAR.xlsx` **reenviado no Telegram**
(2 abas): **≥5k = 21 itens R$251.679** + **<5k = 29 itens R$43.223** = **R$294.902** a fechar. Marcar
coluna SEM_NOTA(S/N). ⚠️ #9 JDD R$14.298 = provável NOTA (EMTECH TELECOM JDD).

**Próximo passo (quando CFO devolver):** juntar S validados + 7 confirmados (gravar em
`customer_semnota_adjust` por ano) → wire no `get_dre` (linhas separadas + CMV por ano). Mecanismo já
existe no `_compute_profit_df_for_period` (linhas sintéticas `_source='semnota'`, api.py:7570) e o DRE
HOJE NÃO inclui sem-nota (é a origem do resíduo R$1.863 RONDOLINK na paridade CMV DRE×Rentab — ver
[[ecommerce-dre-competencia-revisar]] A2). Script de persistência RONDOLINK-style: `scripts/recompute_semnota_snapshot.py` (factor de `customer_revenue_factor`).

> **🔴 PENDENTE (reaberto 18/jun/2026):** incluir o **faturamento sem nota no DRE** de gestão. CFO ainda precisa marcar S/N no XLSX (`receita_sem_nota_REVISAR.xlsx`: 21 ≥R$5k + 29 <R$5k). Depois: separar por ano, aplicar CMV (2024~31%/2025~43%/2026~38%) e lançar a linha "Receita sem documento fiscal" no DRE. **NADA gravado no banco até o CFO fechar.**

**Origem:** o DRE conta receita só de NF-e (`bling_nfe`) + ML. Mas houve **venda recebida no caixa SEM
emissão de NF-e** (clientes que pediram p/ não emitir; revendedores). Está no Sheets como receita
"VENDA DE PRODUTOS" com **favorecido ≠ "X CONNECT IMPORT"** (o X CONNECT IMPORT é o repasse ML).
**Decisão CFO:** lançar essa receita real no DRE de gestão como **"Receita sem documento fiscal"**;
para clientes com nota PARCIAL (ex.: Star2go), lançar só o **excedente** (`recebido − faturado`).

**Método de cruzamento:** receita direta (Sheets, favorecido≠X Connect Import) × NF-e (ambas empresas,
match por nome-núcleo substring). **Imperfeito** (nomes bagunçados/variantes/&) → exige REVISÃO MANUAL.
Excluir intercompany (favorecido "AVIATION") e o canal Havan (faturado à parte).
- Casar por VALOR superestima (pagamento parcial não bate); por TOKEN de 1º nome subestima (colide).
  Substring do nome-núcleo é o melhor, mas ainda erra (Rio Uruguai variante "CENTRAL ARGENTINO" e B&Q
  caíram errado). **O CFO é a autoridade — ele marca S/N.**

**CSV de revisão:** `/home/ubuntu/receita_sem_nota_revisao.csv` (204 linhas: classificacao, favorecido,
recebido, n_receb, nfe_estimado, sem_nota_candidato, CONFIRMA(S/N), obs). Teto bruto candidato
~R$ 1,76M (ZERO R$ 1,25M + PARCIAL R$ 509k, 2024-2026) — **cai muito após remover falsos positivos**.

**Método que funcionou (15/jun):** match por NOME (não por valor — valor dá coincidência). Combinar
substring-core + fuzzy difflib + contenção de tokens, contra NF-e (situacao=6) **E NFS-e (situacao 1,3,
produto E serviço!)**. Sem o filtro de CFOP no match (contava só venda → subcontava NF-e do cliente,
ex.: B&Q). Lista de ≥R$5k caiu de 137 → 21 sem-nota genuínos após o match agressivo. Busca por VALOR
descartada (coincidências: HAVAN R$27k ≠ Produtos Amostra R$27,3k).

**RESULTADO ≥ R$ 5k (15/jun):**
- ✅ Sem-nota CONFIRMADO pelo CFO (grandes): **R$ 615.463** — Israel Teixeira R$159.100 (revendedor),
  Startecno R$117.500, Star2go R$113.085 (2024-25, faturou só a partir de 2026), Thiago Velloso R$70.790,
  Welliton R$51.600, Juliano Pich R$40.480, Augusto Franca R$33.400, Rondolink-excedente R$29.508.
- 📋 A confirmar (21 clientes, verificados por nome — sem nota sob nenhum nome): **R$ 251.679** (CSV
  `sem_nota_revisao_LIMPO.csv` enviado no Telegram). PF/revendedores/empresas pequenas.
- ❌ Faturado (removidos): Rio Uruguai R$126.800, B&Q R$82.889, Marfrig R$16.461 (todos têm nota; o
  match perdia por variante de nome/CFOP — CFO confirmou).
- 🔵 À parte: Produtos Amostra Argentina R$27.300 = EXPORTAÇÃO (sem NF-e nacional, tratamento próprio).
- Cauda < R$ 5k: ~R$ 158k (113 itens, quase tudo PF) — definir CRITÉRIO geral (não item a item).
**Total sem-nota ≥R$5k ≈ R$ 867k** (615k confirmado + 252k a confirmar). CSVs em /home/ubuntu/.

**Revisão manual (placar parcial 15/jun):**
- ✅ SEM-NOTA confirmado: **Israel Teixeira Neto R$ 159.100** (revendedor: vendia p/ terceiros, XConnect
  despachava) · **Welliton Mario R$ 51.600**.
- ❌ FATURADO (descartar): **Rio Uruguai** (~R$59-127k, faturou, muito via Aviation) · **B&Q Energia**
  (R$ 82.889 NF-e, fatura tudo).
- A verificar (prováveis revendedores sem-nota, mesmo padrão do Israel): Startecno (~R$117.500),
  Thiago Velloso (R$70.790), Juliano Pich (R$40.480), Augusto Franca (R$33.400), Star2go-excedente (R$113k).

**✅ CMV do sem-nota — DECISÃO CFO 15/jun: % POR ANO** (não flat). Usar a razão de CMV do **canal ML
retail** de cada ano (= margem de PRODUTO; o sem-nota vendeu a preço de varejo/alto, ex.: Israel):
- **2024: ~31%** (sem ml_orders_sync real; alinha c/ a estimativa ML 2024 e a margem alta do período B2B 77%)
- **2025: ~43,4%** · **2026: ~37,5%** (CMV COMPLETO por canal, validado: total 2026 = R$1.121.101 = DRE).
SEM ajuste premium (CFO optou pelo ML do ano puro). **NÃO** atribuir comissão ML (venda direta), imposto
(sem NF-e) nem overhead (fixo, já no DRE) — só o CMV. Contribuição do sem-nota ≈ receita − CMV(ano).
CMV completo por canal (margem produto): ML 2025 57% / 2026 62%; B2B 2025 77% / 2026 44% (B2B desabou
em 2026 — investigar à parte). Cálculo: replicar inputs do get_dre + compute_profitability, groupby _source.
**Aplicar quando o CFO fechar os 21 do Telegram:** separar receita sem-nota CONFIRMADA por ano →
CMV(ano) → lançar linha "Receita sem documento fiscal" + CMV no DRE de gestão.

**⏳ (antigo) estimar CMV sobre a receita sem-nota — RESOLVIDO acima.**
**⏳ PENDENTE (CFO 15/jun): estimar CMV sobre a receita sem-nota.** Não é lucro puro — os produtos
vendidos tiveram custo. Ao lançar a "Receita sem documento fiscal", aplicar um **CMV estimado** (razão,
ex.: ~31,7% de abril/2025 como no ML estimado [[ecommerce-dre-competencia-revisar]], ou custo por
produto se identificável). Também avaliar imposto (essas vendas sem nota recolheram imposto? provavelmente
não → tratamento à parte). Nada gravado no banco até o CFO fechar a revisão.

## 🔔 LEMBRAR 16/jun/2026 — CFO vai marcar os 21 ≥R$5k (S/N)
**Status 15/jun (fim do dia):** lista pronta, CFO pausou e pediu pra lembrar amanhã. Falta ele marcar
**S** (sem-nota) / **N** (faturado) nos 21 itens ≥R$5k. XLSX `receita_sem_nota_REVISAR.xlsx` no Telegram;
lista também em `/home/ubuntu/sem_nota_limpo.csv`. Cauda <R$5k (29 itens ~R$60k, `sem_nota_abaixo5k_LIMPO.csv`)
AGORA TAMBÉM no mesmo XLSX (15/jun) p/ o CFO olhar junto — 29 itens = R$43.223, marcar S/N ou em bloco. XLSX `receita_sem_nota_REVISAR.xlsx` reenviado no Telegram com 3 abas (Resumo, Revisar>=5k com dicas, Revisar<5k).
- **Varredura por nome (15/jun) contra NF-e+NFS-e das 2 empresas:** só **1 forte candidato a TER nota** →
  **#9 R$14.298 "JDD E T T S LTDA" ↔ EMTECH TELECOM JDD R$14.298 (XConnect, nome+valor exatos)** — provável faturado.
  2 falsos positivos (palavra genérica colidiu): Marvin Particip. (R$18k) e JHLFC (R$7,15k) — seguem sem-nota.
  Os outros 18 sem match real.
- **Próximo passo após o S/N:** somar confirmados + os R$615.463 já fechados → separar por ANO →
  CMV (2024 31% / 2025 43% / 2026 38%) → lançar "Receita sem documento fiscal" + CMV no DRE de gestão.
  **NADA gravado no banco até o CFO fechar.**
