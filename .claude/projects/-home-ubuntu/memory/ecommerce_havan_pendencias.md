---
name: ecommerce-havan-pendencias
description: Incrementos de médio prazo do canal Havan adiados pelo usuário (revisar quando ele perguntar o que está pendente)
metadata: 
  node_type: memory
  type: project
  originSessionId: 8754b4d1-8965-4368-9264-3cf095793a05
---

Pendências de implementação do canal Havan (`ecommerce-agent`), **adiadas pelo usuário em jun/2026** após concluir os 5 itens do roadmap principal (ver [[ecommerce-havan-dashboard]]). **How to apply:** quando o usuário perguntar "o que tem pendente de implementação", revisar esta lista e seguir com a que ele escolher.

**Pendências da simulação de pedido (CFO, 06/jun/2026):**
- **Cadastrar custo exato do Suporte Painel Mini ("case de painel")** — hoje usa CMV APROXIMADO de "Case Slim para vidros curvos" (R$ 35,34, confiança "baixa" no `_HAVAN_CMV_MAP` de api.py; SKU HVNSPPASLMIVE4C). Quando o CFO passar o custo real: criar versão em `product_costs` p/ produto próprio + ajustar `_HAVAN_CMV_MAP` (afeta rentabilidade do canal e /simulacao-havan).
- **Usar o componente EMBALAGEM (já existe, nunca usado)** — fornecedor dará desconto no produto, mas a caixa nova (embalagem definitiva) passa a ser paga pela XConnect. O componente "Embalagem" (component_id=3) JÁ EXISTE em cost_components desde o início, mas NENHUM produto tem linha com ele em product_costs (confirmado 06/jun/2026). **Decisão do CFO: NÃO adicionar componente na simulação** — quando o acordo fechar, cadastrar direto na Gestão de Custos (/custos): linhas component_id=3 + nova versão do componente Produto com o desconto, effective_from na data real. A simulação lê a composição vigente automaticamente. Sem mudança de schema, sem feature nova.

Incrementos de médio prazo (todos com dados já disponíveis, exceto onde notado):
1. **Whitespace por filial** — filiais que vendem a categoria mas têm 0 de um SKU = oportunidade de distribuição/cross-sell. Dado: matriz filial×produto no snapshot.
2. **Dimensão comprador** — capturar `NomeComprador`/`IdComprador` do portal (hoje hardcoded `TODOS`) → quem na Havan compra cada categoria, foco de relacionamento/negociação. **Precisa de captura nova** (variar parâmetro do scraper).
3. **Elasticidade de preço** — track histórico de `preco` (varejo Havan) × velocidade de venda → estimar elasticidade. Habilitado agora pelo histórico de snapshots (#1).
4. **Digest semanal/mensal no Telegram** — resumo do canal: top movers, sugestões de reposição, status da conciliação, risco de devolução.

Outras ideias menos prioritárias já mapeadas: testar `TipoVisaoRelatorio`/`Nivel` ≠ 0 no portal (podem destravar granularidade por loja/margem); detecção de winner acelerando / SKU morrendo via histórico; afinidade produto×região.
