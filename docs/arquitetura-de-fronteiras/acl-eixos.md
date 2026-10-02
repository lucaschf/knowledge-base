# Os 8 eixos de auditoria de uma ACL

Um roteiro para revisar um adapter, gateway ou client de serviço externo. Todos os eixos variam a mesma pergunta: *a borda está traduzindo ou decidindo?*

## Antes de começar: localize a transformação

Identifique o que **entra** (o payload cru do provider) e o que **sai** (a entidade ou o VO de domínio). A auditoria avalia o que acontece entre os dois. Se não há transformação — o adapter só repassa um DTO —, os quatro primeiros eixos quase não se aplicam; concentre-se em erro e observabilidade.

## Os eixos

### 1. Tradução vs. decisão

**Sinal:** `if`, filtro ou ordenação que mantém, descarta, prioriza ou rejeita resultados **pelo conteúdo**.

**Teste:** se eu apagar essa lógica, o adapter ainda faz sentido como tradutor puro? Se sim, a lógica é política e está no lugar errado.

**Correção:** *Move Policy to Use Case*.

### 2. Preservação de sinal

**Sinal:** campos do provider com significado — confiança, "aproximado", "estimado", motivo de recusa — que não viram campo de domínio. Ou dois estados distintos que viram um só.

**Teste:** o caller consegue distinguir os estados que importam **sem reparsear a resposta crua**?

```python
# Sinal colapsado: o prazo some, e o limite de "expresso" vira regra escondida na infra
return ShippingQuote(is_express=raw["days"] <= 2)

# Sinal preservado: o domínio decide o que é expresso, e pode mudar sem tocar no adapter
return ShippingQuote(delivery=DeliveryDays(raw["days"]))
```

**Correção:** *Map Signal to Domain Field*.

### 3. Default seguro (fail-closed)

**Sinal:** caminhos de borda — lista vazia, item único, todos de baixa confiança, campo ausente — sem um default escrito.

**Teste:** em cada um desses caminhos, qual é o default? Ele foi decidido ou é acidental? Quando a resposta vira promessa ao cliente — preço, prazo, disponibilidade —, "sempre devolver algo" é o default errado.

**Correção:** *Replace Fail-Open Default*.

### 4. Proxy semântico e número mágico

**Sinal:** um campo entra no domínio com significado diferente do nome (`has_tracking_code` usado como "foi entregue"). Comparações como `< 2` ou `== "X"` sem um conceito nomeado por trás.

**Correção:** nomear o conceito — um VO, um enum, um método com nome de negócio.

### 5. Acoplamento estrutural

**Sinal:** `zip` ou índice paralelo entre duas listas que precisam ter a mesma ordem e o mesmo tamanho, sem guarda.

**Correção:** carregar o dado junto da entidade na tradução, em vez de recombinar depois com a resposta crua.

### 6. Observabilidade do caminho degradado

**Sinal:** fallbacks que devolvem dado degradado sem log ou métrica próprios.

**Teste:** dá para responder em produção "quantas vezes aceitei dado de baixa confiança por falta de opção?" sem procurar no código?

**Correção:** um evento de log ou contador por caminho degradado, com nome que diga qual é.

### 7. Testabilidade da política

**Sinal:** a única forma de testar uma regra de negócio é mockando a resposta do provider.

**Teste:** a regra roda num teste sem HTTP? Se não, ela está na camada errada — é consequência direta do eixo 1.

### 8. Tradução de erro

**Sinal:** `httpx.HTTPStatusError`, `requests.HTTPError` ou JSON cru subindo para o use case.

```python
# Depois: cada modo de falha vira uma exceção de domínio
try:
    resp = self._request_with_retry("GET", url)
except ProviderAuthError as e:
    raise CatalogProviderMisconfigured() from e
except RetryExhausted as e:
    raise CatalogProviderUnavailable() from e
```

**Correção:** *Translate Provider Error*. O use case decide o que fazer com cada falha sem nunca importar a biblioteca HTTP.

## Calibrar antes de apontar

Um achado precisa de **dano concreto**: um bug provável, uma transação que passa e não deveria, uma mudança que vai doer. "Poderia traduzir melhor" não é achado. "No caso 'só aproximados', isso promete um prazo de entrega para um endereço não confirmado" é.

Uma régua de severidade que funciona:

| Severidade | Quando |
|---|---|
| **Crítico** | Default fail-open num caminho que cobra, promete ou libera algo que não deveria. |
| **Alto** | Política de negócio na borda; sinal de confiabilidade descartado. |
| **Médio** | Proxy semântico documentado, mas frágil; erro mal traduzido, porém contornável. |
| **Baixo** | Acoplamento posicional ou legibilidade, sem dano imediato. |

## Quando o mesmo eixo falha em vários adapters

Aí não é bug pontual — é falta de padrão. Em vez de abrir um card por adapter, registre a regra num ADR. Um candidato que aparece com frequência:

> *"A ACL traduz, não decide. Sinais de confiabilidade do provider viram campos de domínio. O default de borda é fail-closed."*

## Relacionados

- [Anti-Corruption Layer](acl.md)
- [Boolean Blindness](../modelagem-de-dominio/code-smells/boolean-blindness.md)
- [Divergent Change](../modelagem-de-dominio/code-smells/divergent-change.md) — política na borda é Divergent Change no adapter
