---
name: ecommerce-icms-aliquotas-reais
description: "Estrutura matriz RS / filial SP XConnect e alíquotas reais de ICMS — NUNCA simular com rate genérico, sempre olhar ICMS da NF"
metadata: 
  node_type: memory
  type: project
  originSessionId: bf98d7fe-e1e3-46a5-ad40-9ef513875fd4
---

# ICMS XConnect — estrutura de estabelecimentos e alíquotas reais

## Regra de ouro
**Sempre considerar o `valor_icms` que consta na NF-e.** Quando não há NF emitida (invoice_status='not_found' ou sem ml_nfe row), o sistema aplica heurística por par origem→destino, mas o valor real só fecha quando a NF é emitida via Bling e sincronizada.

**Why:** Em 27-28/mai/2026 simulei rentabilidade três vezes seguidas com erro: (1) usei 17,8% (default hardcoded), (2) corrigi para 17% mas ainda como rate flat, (3) implementei regra de origem assumindo SP e calculei 12% para RS, quando a matriz é RS (intra-RS = 17%, não 12%). Usuário corrigiu a cada iteração.

## Estrutura XConnect
- **Matriz:** RS, CNPJ **57.710.730/0001-01** — emite a maioria das NFs ML (9.730 de 11.633 ml_nfe começam com `43`=RS + esse CNPJ)
- **Filial:** SP, CNPJ **57.710.730/0004-46** (confirmado jun/2026: 737 NFs ml_nfe com cUF `35`=SP + esse CNPJ)
- **EBAZAR/ML faturador:** CNPJ **61.053.889/0001-60** (1.068 NFs, todas Aviation/fulfillment — Aviation é Simples, não usa branch de ICMS)
- **Identificação do emitente:** `ml_nfe.invoice_key` (chave de acesso de 44 dígitos). Bytes 0-1 = UF emitente (`43`=RS, `35`=SP), bytes 6-19 = CNPJ emitente.

## Alíquotas reais por par origem → destino

### Origem RS (matriz)
| Destino | Alíquota |
|---|---|
| RS (intra) | **17%** |
| PR, SC, SP, RJ, MG | **12%** (Sul→Sul + Sul→SE) |
| N, NE, CO, ES (demais UFs) | **7%** |

### Origem SP (filial — se aplicável)
| Destino | Alíquota |
|---|---|
| SP (intra) | 18% (alíquota interna SP — ou 0% com ST/diferimento) |
| MG, PR, RJ, RS, SC | 12% |
| N, NE, CO, ES (demais UFs) | 7% |

## ICMS efetivo (saída + DIFAL)
Para vendas B2C interestaduais, ICMS efetivo (saída + DIFAL) **tende a equalizar em ~18%** (alíquota interna da UF destino). Por isso pedidos 66mm XConnect têm margem ~22,4% bastante constante apesar da alíquota saída variar entre 7% e 17%.

## ICMS crédito (por componente)
- Corpo injetado (component_id=1): rate 17% (nacional)
- Imã (component_id=121): rate 4% (importado)
- 66mm: 49,245×17% + 64,92×4% = R$ 10,97/case
- 88mm China: depende de mm_size_id

## ST / diferimento — ICMS saída = 0 na NF
- Substituição Tributária Para Frente: substituto à montante (importador/indústria) já recolheu ICMS de toda a cadeia. XConnect entra como substituída — NF sai com CST 060, `vICMS=0`, sem novo recolhimento.
- Diferimento: ICMS empurrado para etapa posterior. Saída = 0, próxima etapa acumula.
- Quando NF tem ICMS=0 legítimo, o fix abaixo aplica rate por UF mesmo assim (falsa-positiva possível). Detectar CST exigiria adicionar coluna no banco.

## Comissão ML real
NÃO usar 13% genérico em simulações. Validado 27/mai/2026 com pedidos 66mm XConnect R$ 385,76: comissão real **9,91%** (Classic com ajuste de exposed price). Por anúncio varia — sempre olhar `sale_fee / total_amount` em pedidos recentes do MLB específico.

## Fix `metrics.py:261` + `api.py:5836` (28/mai/2026)
**Antes:** `icms_saida = icms_nf if icms_nf > 0 else receita * icms_saida_rate` — aplicava 17% flat em todo pedido ML sem NF, ignorando origem e destino.

**Depois:**
1. `api.py` propaga `n.uf_destino` e `n.invoice_key` para o cálculo. Do `invoice_key`, extrai `uf_emitente` (bytes 0-1).
2. `metrics.py` quando `icms_nf=0`:
   - Se `uf_emitente=RS` e `uf_destino` conhecida: aplica tabela RS→destino acima
   - Caso contrário: mantém fallback no `icms_saida_rate` do tax_config

**Impacto medido XConnect 2026:**
- 3.538 pedidos com NF da ML e ICMS>0 → inalterados (já corretos)
- 22 pedidos com NF da ML mas ICMS=0 e UF conhecida → agora aplicam alíquota origem RS→destino correta
- 34 pedidos sem ml_nfe → seguem fallback genérico (UF desconhecida)

## Fix jun/2026 — branch SP + bug do cUF (RS estava morto)
**Bug descoberto:** `api.py` setava `uf_emitente` com os 2 primeiros dígitos da chave (cUF numérico `"43"`/`"35"`), mas `metrics.py` comparava com `"RS"`/`"SP"` (sigla). Resultado: o branch RS **nunca disparava** desde mai/2026 — caía sempre no `icms_saida_rate` genérico. Impacto era pequeno (XConnect LP 2026 com ICMS=0: só 18 NFs RS) e mascarado porque 2025 era Simples (vai pro branch do Simples via `tax_config_by_ym`).

**Correção:**
1. `api.py` — novo `_IBGE_UF` (cUF→sigla) + helper `_uf_from_invoice_key()`; `uf_emitente` agora guarda sigla. Coerente com `uf_destino` (que já era sigla).
2. `metrics.py` — adicionado branch `_uf_emit == "SP"` com tabela SP→destino (SP=18%, MG/PR/RJ/RS/SC=12%, demais=7%). Afeta 9 NFs XConnect 2026 com ICMS=0.

**Limitação remanescente:**
- DIFAL vem só de `ml_nfe.valor_icms_ufdest`. Pedidos ML interestaduais sem ml_nfe têm DIFAL=0 no cálculo, inflando margem aparente até NF própria ser emitida via Bling e sincronizada.
- Fallback aplica rate por UF mesmo quando ICMS=0 é legítimo (ST/diferimento), tanto p/ RS quanto SP — falsa-positiva possível. Detectar CST exigiria coluna no banco.

Ver [Rentabilidade — modelo e rateio](ecommerce_rentabilidade_pedido.md) e [ICMS model](ecommerce_icms_model.md).
