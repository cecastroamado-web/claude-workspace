# Regras de Consulta Bling API v3 (validadas, aplicar a todas as empresas)

## NF-e de Produto (endpoint GET /nfe)

### Busca
- **NÃO usar filtro de data** da API (`dataEmissaoInicial`/`dataEmissaoFinal`) — bug no boundary do 1º dia do mês perde NF-e
- **Workaround**: buscar TODAS sem filtro de data, paginar com `limite=100&pagina=N`, filtrar `dataEmissao` manualmente no código
- **Valor** só vem no detalhe (`GET /nfe/{id}`), não na listagem — precisa chamar detalhe de cada NF-e

### NF-e válida = tem linkDanfe
- **NÃO confiar no campo `situacao`** da API para determinar se NF-e é válida
- NF-e série 1 (vendas diretas) aparecem com sit=6 na API mesmo estando autorizadas
- Critério correto: **tem `linkDanfe` no detalhe = NF-e válida**
- NF-e canceladas com valor zero simplesmente não aparecem na API — irrelevante

### Situações (referência, mas não usar como filtro principal)
- sit=5: Autorizada (série 2 / ML)
- sit=6: Aparece para série 1 mesmo autorizadas — VERIFICAR linkDanfe
- sit=7: Entrada/remessa (tipo 0, EBAZAR FULL)

### Exclusões para Relatórios de Venda
- Excluir NF-e com destinatário **EBAZAR.COM.BR LTDA** (qualquer CNPJ/filial) — são remessas ML/FULL, não vendas
- Filtrar por nome do contato (`"EBAZAR" in nome.upper()`)

## NFS-e de Serviço (endpoint GET /nfse)

### Busca
- Parâmetro `pagina` é **obrigatório** (diferente do /nfe) — sem ele dá 400
- Usar `pagina=N&limite=100`

### Filtros
- **sit=1**: Autorizada/Emitida — contar
- **sit=3**: Cancelada — ignorar/excluir

### Exclusões para Análise (Aviation específico)
- NFS-e de **GUSTAVO OLIVEIRA DE MACEDO** (R$ 194.849, nov e dez/2025) — ajuste contábil
- Conta na emissão, mas excluir dos relatórios de análise/faturamento

## OAuth Setup
- Criar app em https://developer.bling.com.br/aplicativos
- Escopos necessários: Contatos, Produtos, Pedidos de Venda, Notas Fiscais, NFS-e, Formas de Pagamento, Naturezas de Operação
- Redirect URL: `https://example.com` (capturar code da barra de endereço)
- Code expira em **1 minuto** — trocar imediatamente
- Alterar escopos **revoga tokens** — refazer OAuth
- Script: `setup_bling_oauth.py` ou troca manual via POST /oauth/token
- Token: access 6h, refresh 30 dias (auto-renova)

## XConnect — Regra Especial de Faturamento

### Problema
- XConnect tem 2 lojas ML: Normal (loja_id=205212252) e FULL (loja_id=205258337)
- NF-e do FULL são emitidas pelo Mercado Livre, **NÃO importam para o Bling `/nfe`**
- Mesmo NF-e vinculada a pedido (`notaFiscal.id != 0`) não aparece na listagem `/nfe`
- Chamado aberto no Bling (15/mar/2026) para resolver importação

### Data de corte: 16/mar/2026
- Em 15/mar/2026 o usuário ativou integração ML→Bling para importar NF-e automaticamente
- Usar **16/mar/2026** como corte (parte do dia 15 pode não ter exportado)

### Antes de 16/mar/2026 (regra híbrida, evita dupla contagem)
- **Vendas ML** → endpoint `GET /pedidos/vendas`, filtrar por `loja.id` (Normal + FULL)
- **Vendas diretas (fora ML)** → endpoint `GET /nfe` **apenas série 1** (`chaveAcesso[22:25] == "001"`)
- **NÃO incluir NF-e série 2** do `/nfe` — já estão contabilizadas nos pedidos de venda
- Série 1: 506 NF-e = R$ 2.505.621 | Série 2 no /nfe: 67 = R$ 23.928 (descartadas, já nos pedidos)

### A partir de 16/mar/2026 (regra padrão, igual Aviation)
- **Todas as vendas** → endpoint `GET /nfe` com regra padrão (linkDanfe, excluir EBAZAR)
- Não precisa mais consultar pedidos de venda para faturamento

### Lojas ML XConnect
- `205212252` = ML Normal (835 pedidos, R$ 840k acumulado)
- `205258337` = ML FULL (10.178 pedidos, R$ 4.95M acumulado)
- FULL começou dez/2024, domina volume desde jan/2025

### XConnect NF-e diretas (Bling /nfe)
- 557 NF-e válidas desde nov/2024, zero EBAZAR
- NF-e série 1 = vendas B2B diretas
- NF-e série 2 = poucas, ML Normal (quando Bling emite)

### Aviation — NF-e normais
- Aviation usa apenas `/nfe` — NF-e do FULL aparecem normalmente no Bling
- EBAZAR = remessas ao FULL, excluir de relatórios de venda

## Validações Realizadas (14-15/mar/2026) — Aviation
- Jan/2026 NF-e: 62 = R$ 163.421,27 ✅ (centavo)
- Fev/2026 NF-e: 87 = R$ 110.492,34 ✅ (centavo)
- Jan/2026 NFS-e: 8 = R$ 3.070 ✅
- Fev/2026 NFS-e: 1 = R$ 8.000 ✅
- Mar/2026 NFS-e: 16 = R$ 29.250 ✅
- Nov/2025 NFS-e: 3 = R$ 207.349 ✅
- Dez/2025 NFS-e: 2 = R$ 10.850 ✅ (análise, sem Gustavo)
- Histórico completo ago/2025-mar/2026 validado

## Validações Realizadas (15/mar/2026) — XConnect
- Pedidos ML: 11.013 total, volumes fazem sentido (confirmado pelo usuário)
- Jan/2026 ML: 645 pedidos = R$ 309.052 (12 Normal + 633 FULL)
- NF-e diretas: 557 desde nov/2024, R$ 2.539.717 total
- XConnect não tem módulo NFS-e (403 esperado)
