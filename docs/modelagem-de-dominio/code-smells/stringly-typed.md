# Stringly-Typed Code

**Princípio violado:** estado ilegal representável — uma `str` carrega um conjunto fechado de valores válidos, mas nada impede um valor fora do conjunto.

## O que é

Usar `str` (ou `int` mágico) para representar algo que tem um domínio fechado e conhecido: status, tipo, categoria, papel. É um primo do [Primitive Obsession](primitive-obsession.md), mais específico.

## Como detectar

- Campos `status: str`, `type: str`, `kind: str` comparados com literais espalhados (`if status == "paid"`).
- Strings mágicas repetidas pelo código, sujeitas a typo silencioso (`"payed"`).
- Lógica que faz `match`/`if` sobre valores de string sem exaustividade garantida.

## Refactoring

**Replace String with Enum** (ou VO quando há comportamento associado). O conjunto de valores vira fechado e o type checker passa a cobrar exaustividade.

## Antes / depois

```python
# Antes
def can_ship(status: str) -> bool:
    return status == "paid"  # typo aqui não falha em tempo de tipo
```

```python
# Depois
from enum import Enum

class OrderStatus(Enum):
    PENDING = "pending"
    PAID = "paid"
    CANCELLED = "cancelled"

def can_ship(status: OrderStatus) -> bool:
    return status is OrderStatus.PAID
```

## Quando NÃO marcar

- Texto livre de verdade (nome, descrição) — aí `str` é o tipo certo.
- Valor que vem cru de um sistema externo e ainda não cruzou a fronteira do domínio.

## Relacionados

- [Primitive Obsession](primitive-obsession.md)
- [Boolean Blindness](boolean-blindness.md)
