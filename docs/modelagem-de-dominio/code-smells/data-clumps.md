# Data Clumps

**Princípio violado:** coesão / acoplamento — o mesmo grupo de dados anda sempre junto, mas não foi reconhecido como um conceito.

## O que é

Três ou mais campos/parâmetros que aparecem juntos repetidamente (em assinaturas, atributos, dicts). Se andam sempre juntos, provavelmente **são** um conceito que ainda não tem nome.

## Como detectar

- `def f(street, number, city, zip_code)` repetido em vários lugares.
- O mesmo trio `latitude, longitude, accuracy` espalhado.
- Quando você adiciona um campo ao grupo, precisa editar todas as assinaturas.

## Refactoring

- **Introduce Parameter Object / Extract Value Object** — o grupo vira um tipo (`Address`, `GeoPoint`).
- Bônus: o novo tipo vira lar natural para o comportamento que antes era [Feature Envy](feature-envy.md).

## Antes / depois

```python
# Antes
def register(name, street, number, city, zip_code): ...
def update(user_id, street, number, city, zip_code): ...
```

```python
# Depois
@dataclass(frozen=True)
class Address:
    street: str; number: str; city: str; zip_code: str

def register(name, address: Address): ...
def update(user_id, address: Address): ...
```

## Quando NÃO marcar

- Dois parâmetros que coincidem uma vez só — clump precisa de **repetição**.
- Parâmetros que andam juntos por acaso, sem relação conceitual.

## Relacionados

- [Primitive Obsession](primitive-obsession.md)
- [Feature Envy](feature-envy.md)
