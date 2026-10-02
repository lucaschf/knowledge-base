# Anti-Corruption Layer

A Anti-Corruption Layer (ACL) é a camada que **traduz** o modelo de um sistema que você não controla para o modelo do seu domínio. Ela existe para que a forma, os nomes e as esquisitices do provider não se espalhem pelo código.

A definição é conhecida. O erro que ela permite cometer é menos comentado.

## A borda traduz, não decide

> O adapter responde **"o que o provider disse"**. O use case e o domínio respondem **"o que a gente faz com isso"**.

Quando a borda responde a segunda pergunta, três coisas acontecem ao mesmo tempo:

1. **A regra fica invisível.** Quem lê o domínio não a encontra, porque ela mora num arquivo de infraestrutura.
2. **A regra fica intestável sem rede.** O único jeito de exercitar a decisão é mockando a resposta HTTP.
3. **O caso-limite é resolvido pelo default mais conveniente**, que quase sempre é o permissivo.

O terceiro ponto é o mais perigoso, e merece um nome.

## Fail-closed: na dúvida, não passa

Toda borda tem caminhos degradados: lista vazia, campo ausente, todos os resultados de baixa confiança. Em cada um deles existe um default — e a pergunta é se ele foi **escrito** ou se é **acidental**.

Quando o resultado vira uma promessa — um preço cobrado, um prazo de entrega, um acesso liberado —, o default precisa ser **fail-closed**: na ausência de sinal confiável, o sistema não promete — ele recusa ou pede confirmação. "Sempre devolver alguma coisa" não pode ganhar de "nunca devolver dado não-confiável como se fosse confiável".

## Um exemplo: a borda que decide

Um provider de endereço por CEP devolve candidatos, cada um com um campo `precision`: `exact` ou `approximate`.

```python
# Antes — a política mora no adapter, e o fallback é permissivo
class AddressAdapter(AddressLookup):
    def lookup(self, cep: str) -> list[Address]:
        raw = self._client.get(f"/cep/{cep}")["results"]
        exact = [r for r in raw if r["precision"] == "exact"]
        chosen = exact or raw            # nenhum exato? devolve os aproximados...
        return [self._to_address(r) for r in chosen]   # ...sem dizer que são aproximados
```

Três problemas numa linha só. A regra "preferir exato" é política e está na infra. O sinal `precision` é descartado na tradução. E, quando só há aproximados, o caller recebe endereços aproximados **indistinguíveis** de exatos — e o domínio promete um prazo de entrega para um endereço que ninguém confirmou.

```python
# Depois — a borda preserva o sinal; a política fica explícita no domínio
@dataclass(frozen=True)
class Address:
    street: str
    city: str
    is_exact: bool               # o sinal do provider vira campo de domínio

class AddressAdapter(AddressLookup):
    def lookup(self, cep: str) -> list[Address]:
        raw = self._client.get(f"/cep/{cep}")["results"]
        return [self._to_address(r) for r in raw]       # devolve todos, com a flag

class ResolveDeliveryAddress:
    def __call__(self, candidates: list[Address]) -> Address:
        exact = [a for a in candidates if a.is_exact]
        if len(exact) == 1:
            return exact[0]
        raise AddressNeedsConfirmation(candidates)      # caso-limite explícito: fail-closed
```

A política agora roda num teste sem HTTP, o caso "só aproximados" virou uma decisão visível, e o sinal do provider chega inteiro a quem decide.

## Um caso real: a ACL do TMDB no HomeFlix

O HomeFlix consome o TMDB por uma ACL dividida em dois arquivos: um cliente HTTP fino (autenticação, retry, rate limit) e um [mapper puro](https://github.com/lucaschf/homeflix/blob/develop/src/modules/metadata/infrastructure/tmdb_response_mapper.py) que transforma o JSON nos DTOs do módulo, sem fazer nenhuma chamada de rede. A separação é o que deixa a tradução testável sem rede.

O mapper tem lógica de escolha — e é um bom lugar para ver onde fica a linha:

- **`pick_best_logo_url`** escolhe, entre vários logos, o do idioma pedido. Isso é tradução: o caller pediu `pt-BR`, e a borda devolve a melhor representação *do mesmo fato* nesse idioma. Não há consequência de negócio em jogo.
- **`pick_year_match`** escolhe, entre resultados de busca, o do ano informado. Quando nenhum bate, devolve `None` em vez do primeiro resultado. A docstring diz por quê: para não devolver silenciosamente um título popular de outro ano. É um default fail-closed escrito na borda.

E a reconciliação — decidir se o dado do TMDB sobrescreve o que já está salvo — **não** está no mapper. Ela vive na camada de aplicação, e o [ADR-025](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-025-provider-metadata-reconciliation-in-application.md) registra por que ela também não foi para dentro da entidade.

## Para auditar uma ACL

A auditoria tem oito eixos: tradução vs. decisão, preservação de sinal, default seguro, proxy semântico, acoplamento estrutural, observabilidade do caminho degradado, testabilidade da política e tradução de erro. Cada um com o sinal detectável e o refactoring que corrige: [os 8 eixos](acl-eixos.md).

## Relacionados

- [Context map](../ddd-estrategico/context-map.md)
- [Boolean Blindness](../modelagem-de-dominio/code-smells/boolean-blindness.md) — o que acontece quando um score vira `bool` na borda
- [Primitive Obsession](../modelagem-de-dominio/code-smells/primitive-obsession.md)
