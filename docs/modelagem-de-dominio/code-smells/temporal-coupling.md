# Temporal Coupling

**Princípio violado:** estado ilegal representável — métodos só funcionam se chamados numa ordem específica, mas o tipo não impede a ordem errada.

## O que é

A corretude depende da **sequência** de chamadas, e nada no tipo garante essa sequência. Chamar `b()` antes de `a()` compila, mas quebra em runtime (ou pior, silenciosamente).

## Como detectar

- Objeto com método `configure()`/`init()`/`prepare()` que precisa rodar antes dos outros.
- Atributos que começam `None` e só ficam válidos depois de um setup.
- Comentários do tipo `# chame setUp() primeiro`.

## Refactoring

- **Construção completa** — o objeto nasce válido pelo construtor; sem passo de setup separado.
- **Builder / factory** quando a montagem é complexa.
- **State machine explícita** quando a ordem é mesmo essencial ao domínio — cada estado só expõe as transições válidas.

## Antes / depois

```python
# Antes
checkout = Checkout()
checkout.set_cart(cart)    # se esquecer, explode lá na frente
checkout.confirm()
```

```python
# Depois
checkout = Checkout.for_cart(cart)  # nasce pronto e válido
checkout.confirm()
```

## Quando NÃO marcar

- Ordem natural e inevitável do protocolo (abrir → ler → fechar) já encapsulada por um context manager.

## Relacionados

- [Anemic Domain Model](anemic-domain-model.md) — anemia costuma vir acompanhada de setup temporal.
