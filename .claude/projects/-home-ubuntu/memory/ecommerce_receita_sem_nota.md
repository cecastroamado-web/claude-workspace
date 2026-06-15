---
name: ecommerce-receita-sem-nota
description: ecommerce-agent — auditoria de RECEITA SEM DOCUMENTO FISCAL (vendas recebidas sem NF-e) p/ o DRE de gestão; revisão manual em curso
metadata:
  node_type: memory
  type: project
  originSessionId: bc097805-a0ae-41ef-988e-5a1b9b9d8aab
---

# Receita sem documento fiscal (vendas sem NF-e) — auditoria p/ DRE (CFO 15/jun/2026)

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
