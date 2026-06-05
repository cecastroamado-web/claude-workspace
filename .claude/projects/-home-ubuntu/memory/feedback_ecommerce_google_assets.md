---
name: feedback-ecommerce-google-assets
description: "ecommerce-agent: TODA planilha/Drive/asset Google criado para o serviço deve ser de propriedade da conta xconnectimport@gmail.com, nunca da conta pessoal cecastroamado@gmail.com."
metadata: 
  node_type: memory
  type: feedback
  originSessionId: bd041687-fc17-4612-b668-4954b8538c30
---

# Padrão: assets Google do ecommerce-agent pertencem a xconnectimport@gmail.com

Toda planilha do Google Sheets, pasta do Drive ou arquivo Google criado para uso pelo `ecommerce-agent` (qualquer coisa que o backend leia ou escreva via API) **deve ser criado já como propriedade da conta de serviço `xconnectimport@gmail.com`** — nunca da conta pessoal `cecastroamado@gmail.com`.

**Why:** o serviço autentica com o token OAuth da `xconnectimport@gmail.com` (ver [[ecommerce-oauth-tokens]]). Quando o asset é criado em outra conta, o backend devolve 404/403 mesmo com token válido — exatamente o que aconteceu com a planilha B2B em mai/2026 (criada na conta pessoal, endpoint quebrado até o usuário compartilhar manualmente). É um modo de falha silencioso e recorrente.

**How to apply:**
- Antes de criar qualquer Sheet/Drive novo via API, confirme que o token em uso é da `xconnectimport@gmail.com` (use `drive.about().get(fields='user')` se houver dúvida).
- Se for orientar o usuário a criar manualmente no browser, peça explicitamente para criar logado na `xconnectimport@gmail.com` — não no Gmail pessoal.
- Se já existe um asset na conta errada: pedir Compartilhamento + Transferência de Propriedade pra `xconnectimport@gmail.com`, não apenas compartilhar.
- Aplica-se ao ecommerce-agent. Outros projetos (email-agent, b3analyzer, etc.) podem ter convenções diferentes — confirmar antes de generalizar.
