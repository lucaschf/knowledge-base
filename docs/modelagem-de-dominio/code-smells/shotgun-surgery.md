# Shotgun Surgery

**Princípio violado:** coesão / acoplamento — uma única mudança conceitual obriga a editar muitos arquivos espalhados.

## O que é

Para fazer **uma** alteração de comportamento, você precisa tocar em N lugares diferentes. O conhecimento sobre aquela regra está difuso, não centralizado. É o oposto do [Divergent Change](divergent-change.md).

## Como detectar

- "Pra mudar o formato do CEP tive que mexer em 7 arquivos."
- Regra de negócio duplicada/espalhada em validators, serializers, services.
- Medo de mudar algo simples porque "esquece um lugar e quebra".

## Refactoring

- **Move Method / Move Field** — juntar o que está espalhado num lugar só.
- **Inline + Extract** pra consolidar a regra duplicada num único ponto (um VO costuma ser o destino).

## Antes / depois

A regra "CEP tem 8 dígitos" repetida no request validator, no service, no repositório e no relatório → centralizada num único `Cep` ([Value Object](primitive-obsession.md)). Mudou a regra, muda num lugar.

## Quando NÃO marcar

- Mudança que é grande **por natureza** (renomear um conceito central) — aí espalhar é inevitável, não é smell.

## Relacionados

- [Divergent Change](divergent-change.md) — o espelho deste smell.
- [Primitive Obsession](primitive-obsession.md) — causa comum de Shotgun Surgery.
- [Integration strength](../../acoplamento/integration-strength.md) — no nível de módulo, Shotgun Surgery costuma ser *functional coupling* simétrico.
