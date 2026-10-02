# Boolean Blindness / Flag Arguments

**Princípio violado:** estado ilegal representável — um `bool` não diz, no ponto de chamada, o que `True` significa; e flags permitem combinações sem sentido.

## O que é

Dois sintomas relacionados:

- **Boolean blindness:** `process(True, False, True)` — o leitor não tem ideia do que cada flag controla.
- **Flag argument:** um parâmetro booleano que faz a função se comportar de dois jeitos diferentes (geralmente esconde duas funções numa).

## Como detectar

- Chamadas com booleanos posicionais: `notify(user, True, False)`.
- `if flag:` logo no início do corpo, dividindo a função em dois caminhos quase independentes.
- Vários booleanos no construtor que permitem estados impossíveis (`is_active=True, is_deleted=True`).

## Refactoring

- **Replace flag with explicit method** — `enable()` / `disable()` em vez de `set_active(bool)`.
- **Replace boolean with enum** quando há mais de dois estados ou o `True/False` não é auto-explicativo.
- **Keyword-only arguments** quando o booleano precisa mesmo existir: `def notify(user, *, urgent: bool)`.

## Antes / depois

```python
# Antes
create_user(name, email, True, False)  # o que é True? e False?
```

```python
# Depois
create_user(name, email, status=UserStatus.ACTIVE)
```

## Quando NÃO marcar

- Booleano único, keyword-only e auto-explicativo (`dry_run=True`) numa função simples.

## Relacionados

- [Stringly-Typed Code](stringly-typed.md)
- [Primitive Obsession](primitive-obsession.md)
- [Os 8 eixos de auditoria de ACL](../../arquitetura-de-fronteiras/acl-eixos.md#2-preservacao-de-sinal) — quando um score do provider vira `bool` na borda.
