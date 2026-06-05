---
name: Mídia Brasil — migração para canal Internacional via Reweb Corp (mai/2026)
description: A queda do CMV Mídia na DRE Brasil em 2026 é mudança contábil — mídia migrou para cartão internacional pago via wire→Reweb Corp, NÃO saída de clientes
type: project
originSessionId: c738a5e8-9708-47a0-9874-623b8cc12be8
---
A queda do gasto Mídia na DRE Brasil de R$ 4,68M (2024) → R$ 0,62M (2026 parcial) **não é saída de clientes**. É mudança de canal contábil:
- Antes (até 2025): mídia de clientes Brasil era paga parte em BRL na conta Brasil, parte via cartão internacional → wire para Reweb Corp.
- A partir de 2026: quase tudo migrou para o cartão internacional → wire→Reweb. A fração em BRL no Brasil ficou pequena.

**Why:** a primeira leitura ("saída de 104 clientes") foi corrigida pelo usuário. Os clientes continuam ativos; só o canal de pagamento mudou. A Reweb Corp recebe wires de todos os países e paga FB/Google/TikTok centralizadamente.

**Quadro consolidado (Mídia Brasil + Wire→Reweb global, R$ M):**
- 2024: 4,68 + 3,15 = 7,83
- 2025: 3,10 + 3,38 = 6,48 (-17,3%)
- 2026 anualizado: 1,53 + 4,33 = 5,86 (-25,1%)

A queda real consolidada é -25%, não -67%/-86%.

**Wire→Reweb média mensal:** 2024=R$ 263k · 2025=R$ 281k · **2026=R$ 361k** (+~R$ 80-100k/mês = aumento que absorve a fração migrada do Brasil).

**How to apply:**
- DRE Brasil como está subestima o CMV em 2026 → a Margem Bruta Brasil aparece artificialmente boa.
- O `wire_midia` no aggregate_monthly mistura TODOS os países (Colombia, México, Equador, Brasil) — não dá pra isolar Brasil sem quebra que só a Reweb Corp tem.
- Para corrigir, opções: (a) alocação proporcional por receita de mídia BQ; (b) pedir quebra mensal à Reweb Corp; (c) caption no dashboard explicando.
- Argentina paga direto (lançamentos "Google - Inserção" no extrato local), não via Reweb.
- Lançamentos R$ 9.960/mês jun→dez/2026 sem faturamento correspondente — provavelmente SaaS recorrente, verificar.

Doc: `docs/diagnostico_queda_midia.md` (10/mai/2026, versão revisada).
