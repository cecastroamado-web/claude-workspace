---
name: adaptive-thinking-signature
description: "Adaptive thinking + tool_use no Anthropic SDK exige que thinking blocks devolvam o campo `signature` ao continuar a conversa (loop tool_use). Sem ele a API retorna 400."
metadata: 
  node_type: memory
  type: project
  originSessionId: 1f222532-0672-4e55-afdd-cd9965623b68
---

# Adaptive thinking + tool_use: signature é obrigatória

Quando uma chamada usa `thinking={"type": "adaptive"}` (Sonnet 4.6 / Opus 4.7) e o
modelo decide pensar antes de chamar uma tool, a resposta vem com um bloco
`thinking` que carrega o campo **`signature`** (assinatura criptográfica do
raciocínio). Esse bloco precisa voltar **idêntico** para a API no próximo turno
do loop tool_use — incluindo a signature — senão a API rejeita com:

```
400 invalid_request_error
messages.N.content.0.thinking.signature: Field required
```

**Why:** Bug observado em produção no ecommerce-agent em 2026-05-17 quando o
Carlos perguntou "como estamos de venda esse mês?" no Telegram. Sonnet 4.6 com
adaptive thinking respondeu com `[thinking, tool_use(get_revenue)]`, o
`_serialize_content` em `main.py` extraía `{"type": "thinking", "thinking":
block.thinking}` mas descartava `signature` → 400 no segundo turno do loop.

**How to apply:**

1. Ao serializar blocks do SDK para reenviar à API (loop tool_use, fork de
   conversa, etc.), **preserve `signature`** em blocos `thinking` e o campo
   `data` em blocos `redacted_thinking`. Padrão:

   ```python
   elif block_type == "thinking":
       out = {"type": "thinking", "thinking": getattr(block, "thinking", "")}
       sig = getattr(block, "signature", None)
       if sig:
           out["signature"] = sig
       result.append(out)
   elif block_type == "redacted_thinking":
       data = getattr(block, "data", None)
       if data is not None:
           result.append({"type": "redacted_thinking", "data": data})
   ```

2. **Não persistir thinking blocks** em armazenamento de longo prazo (SQLite,
   Mem0, arquivos de memória). A signature é válida apenas dentro da
   conversation original; reidratar em sessão futura faz a API rejeitar.
   Filtre `thinking` / `redacted_thinking` antes de gravar.

3. **Filtro defensivo no reidrate**: mesmo com (2), se houver dados legados
   no SQLite, filtre `thinking` / `redacted_thinking` no caminho de leitura
   (`_decode_content_from_storage` em `memory_store.py`).

**Onde isso aparece no workspace:**

- ecommerce-agent (corrigido em mai/2026): [[autoria adaptive thinking sonnet]]
- email-agent: hoje ainda usa `thinking={"type": "enabled", "budget_tokens": 2048}`
  (estilo antigo, sem signature). Se migrar para `{"type": "adaptive"}`, aplicar
  todas as 3 regras acima.
- followup-agent / b3analyzer / dividend-portfolio: não usam tool_use loop
  multi-turn hoje, mas se forem nessa direção, mesma armadilha.
