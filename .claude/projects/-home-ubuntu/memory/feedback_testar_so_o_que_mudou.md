---
name: feedback-testar-so-o-que-mudou
description: Validar apenas o que foi alterado; não testar fluxos não relacionados ou ainda não finalizados
metadata: 
  node_type: memory
  type: feedback
  originSessionId: cdca63bc-a576-479d-afdf-799ffd73b806
---

Ao concluir uma mudança, o teste de verificação deve cobrir SÓ o que mudou — não fluxos vizinhos que não foram tocados. Em 07/jun/2026, após corrigir CI + restart do ecommerce-agent, propus testar um pedido completo via Telegram; o usuário corrigiu: "por que testar isso se não trabalhamos nisso e na realidade esse processo ainda não foi finalizado e validado".

**Why:** testar fluxo não relacionado não prova nada sobre a mudança feita, consome tempo do usuário e, pior, pode disparar efeitos reais (NF-e, boleto) num processo que nem está validado.

**How to apply:**
- Mapear verificação 1:1 com o diff: endpoint corrigido → smoke test daquele endpoint; lint/testes → CI verde; restart → systemctl status + /health.
- O pipeline de pedido via Telegram do ecommerce-agent (parse → Bling NF-e → Inter boleto → Sheets) **ainda não foi finalizado/validado** (status em jun/2026) — não tratar como fluxo de produção estável nem usar como prova de saúde do serviço. Ver [[ecommerce-ci-github]].
