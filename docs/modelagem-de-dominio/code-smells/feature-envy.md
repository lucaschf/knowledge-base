# Feature Envy

**Princípio violado:** comportamento longe dos dados (_Tell, Don't Ask_) — um método se interessa mais pelos dados de outro objeto do que pelos do seu próprio.

## O que é

Um método que acessa repetidamente os dados de **outro** objeto pra fazer seu trabalho. O comportamento está no lugar errado: deveria morar onde os dados estão.

## Como detectar

- Método que chama `outro.get_a()`, `outro.get_b()`, `outro.get_c()` e combina tudo.
- Use case que extrai vários campos de uma entity só pra calcular algo que é regra daquela entity.
- A "inveja" é literal: o método quer as features de outra classe.

## Refactoring

- **Move Method** — mover o método pra classe cujos dados ele inveja.
- **Extract + Move** quando só um trecho tem a inveja.

## Antes / depois

```python
# Antes — o service inveja os dados de Address
class ShippingService:
    def cost(self, order):
        addr = order.address
        if addr.country == "BR" and addr.region == "N":
            return 50
        return 20
```

```python
# Depois — o cálculo mora onde os dados estão
class Address:
    def shipping_zone(self) -> Zone:
        if self.country == "BR" and self.region == "N":
            return Zone.REMOTE
        return Zone.STANDARD
```

## Quando NÃO marcar

- Use cases **devem** orquestrar e ler de vários objetos — isso é o trabalho deles, não inveja.
- Acesso pontual a um getter não é Feature Envy; o sinal é a _concentração_ de acessos a um objeto alheio.

## Relacionados

- [Anemic Domain Model](anemic-domain-model.md)
- [Data Clumps](data-clumps.md)
