# Anemic Domain Model

**Princípio violado:** comportamento longe dos dados (_Tell, Don't Ask_) — entities viram sacos de dados com getters/setters, e a regra de negócio mora em services de fora.

## O que é

Objetos de domínio que só guardam estado, sem comportamento. Toda a lógica que deveria ser deles vive em "services" que leem os dados, decidem, e escrevem de volta. É DDD na pasta, não na modelagem.

## Como detectar

- Entity só com `@property`/setters e nenhum método de negócio.
- Service que faz `entity.get_x()`, calcula algo, e chama `entity.set_y(...)`.
- Invariantes checadas no use case, não na entity.

## Refactoring

- **Move Method** — empurrar a regra pra dentro da entity que tem os dados.
- A entity passa a **proteger suas invariantes**: em vez de `set_status`, expõe `pay()`/`cancel()` que só transicionam de forma válida.

## Antes / depois

```python
# Antes — anêmico
class Order:
    def __init__(self): self.status = "pending"
    def get_status(self): return self.status
    def set_status(self, s): self.status = s

class OrderService:
    def pay(self, order: Order):
        if order.get_status() != "pending":
            raise ValueError(...)
        order.set_status("paid")
```

```python
# Depois — a regra mora na entity
class Order:
    def __init__(self): self._status = OrderStatus.PENDING
    def pay(self) -> None:
        if self._status is not OrderStatus.PENDING:
            raise InvalidTransition(self._status)
        self._status = OrderStatus.PAID
```

## Quando NÃO marcar

- DTOs, read models e projeções **devem** ser anêmicos — são dados de transporte, não domínio.
- Orquestração entre vários agregados é legítima no use case; isso não é anemia.

## Relacionados

- [Feature Envy](feature-envy.md) — a lógica anêmica no service quase sempre é Feature Envy pela entity.
- [ADR-025 do HomeFlix](../../estudos-de-caso/homeflix/decisoes.md#adr-025-uma-recomendacao-de-auditoria-recusada) — um caso em que mover a regra para a entidade foi recusado, com argumento.
