# Primitive Obsession

**Princípio violado:** estado ilegal representável — a invariante "isto é um e-mail/preço/token válido" vira disciplina de runtime em vez de garantia do tipo.

## O que é

Usar tipos primitivos (`str`, `int`, `float`, `dict`) para conceitos de domínio que têm regras próprias.

## Como detectar

- Assinatura com primitivo onde existe um conceito: `def send_invoice(email: str)`.
- A mesma validação do primitivo repetida em vários pontos.
- Conjunto de constantes que formam um conceito (`MIN_DISCOUNT = 0.05`, `MAX_DISCOUNT = 0.5`).
- Dá pra trocar dois parâmetros de mesma ordem sem o type checker reclamar.

## Refactoring

**Extract Value Object** — encapsular o primitivo num tipo `frozen` que valida na construção. A invariante passa a ser garantida pelo tipo: se você tem um `Email`, ele é válido por definição.

## Antes / depois

```python
# Antes — a validade do e-mail é disciplina de runtime, repetida em todo lugar
def send_invoice(email: str) -> None:
    if "@" not in email or email.startswith("@"):
        raise ValueError("e-mail inválido")
    ...
```

```python
# Depois — o tipo carrega a invariante
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if "@" not in self.value or self.value.startswith("@"):
            raise ValueError("e-mail inválido")

def send_invoice(email: Email) -> None:  # se chegou aqui, é válido
    ...
```

## Quando NÃO marcar

- Primitivo sem regra própria (contador de loop, um índice).
- Na borda (DTO de request): primitivos são normais; a invariante nasce ao **cruzar** pro domínio.
- Conceito de uso único e trivial onde a indireção do VO não paga.

## Relacionados

- [Stringly-Typed Code](stringly-typed.md) — caso particular com `str` de domínio fechado.
- [Data Clumps](data-clumps.md) — quando vários primitivos andam sempre juntos.
- [ADR-018 do HomeFlix](../../estudos-de-caso/homeflix/decisoes.md#adr-018-identificadores-como-value-objects-nas-fronteiras) — o dano real de ids como `str`: *default-deny silencioso*.
