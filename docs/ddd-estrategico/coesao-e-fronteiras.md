# Coesão e problemas de fronteira

Uma fronteira está boa quando o que está dentro dela **pertence junto**. Coesão é a forma de medir isso — de um jeito grosseiro, mas útil o bastante para comparar alternativas.

## Um score de coesão em quatro dimensões

Para um grupo de conceitos que você acha que formam um contexto:

| Dimensão | Pergunta | Pontos |
|---|---|---|
| **Linguística** | Compartilham vocabulário? | 0–3 |
| **De uso** | Aparecem juntos nos mesmos use cases? | 0–3 |
| **De dados** | Têm relação direta entre si? | 0–2 |
| **De mudança** | Mudam juntos no histórico do git? | 0–2 |

```
coesão = (linguística + uso + dados + mudança) / 10

8–10 alta · 5–7 média · 0–4 baixa
```

A dimensão de mudança é a única que vem de dados objetivos. Um jeito simples de medir é contar quais módulos aparecem juntos no mesmo commit:

```bash
git log --since="6 months ago" --format="%H" --no-merges | while read c; do
  git show --format="" --name-only "$c" | grep "^src/modules/" | cut -d/ -f3 | sort -u | paste -sd "+" -
done | grep "+" | sort | uniq -c | sort -rn | head -15
```

Pares que aparecem muito juntos têm coesão de mudança alta — o que pode significar que pertencem ao mesmo contexto, ou que existe um acoplamento que ninguém declarou.

## Os cinco problemas de fronteira mais comuns

| Problema | Como aparece | O que fazer |
|---|---|---|
| **Mistura linguística** | Vocabulários diferentes no mesmo módulo: credenciais e regras de cobrança no mesmo service. | Separar em contextos distintos. |
| **Generic dentro do core** | Envio de e-mail no meio do cálculo de preço. | Extrair para um subdomínio genérico. |
| **Conceito sem dono** | Uma entidade usada por três contextos, com significados embaralhados. | Clarificar: provavelmente são dois ou três conceitos com o mesmo nome. |
| **Responsabilidades mistas** | Uma classe que atende dois negócios. | Dividir por subdomínio. |
| **Fronteira por camada técnica** | O "contexto" é uma camada inteira. | Redesenhar por capacidade de negócio. |

## Antes de propor uma divisão

Nem toda coesão baixa é problema. Três perguntas antes de recomendar dividir:

1. **A divisão tem valor de negócio que dá para nomear?** "Os vocabulários são diferentes" não basta. Diga o dano: o time confunde os conceitos, uma mudança arrasta a outra, o modelo não se explica para quem conhece o negócio.
2. **O tamanho justifica?** Num monólito modular mantido por uma pessoa, dois subdomínios pequenos e estáveis podem viver juntos por decisão consciente.
3. **É generic estável?** Subdomínios genéricos têm coesão menor por natureza. Não refatore o que não dói.

!!! tip "Mudança de fronteira é cara: faça devagar e com ADR"
    Redesenhar uma fronteira toca dados, contratos e testes. A forma segura é incremental (Strangler Fig, um pedaço por vez) e sempre precedida de um ADR. O [ADR-032 do HomeFlix](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-032-decompose-media-into-subdomains.md) é um exemplo: extrai streaming, metadados e marcadores do catálogo, um por PR, cada um com seu enabler.

## Relacionados

- [Bounded contexts](bounded-contexts.md)
- [Shotgun Surgery](../modelagem-de-dominio/code-smells/shotgun-surgery.md) e [Divergent Change](../modelagem-de-dominio/code-smells/divergent-change.md) — os sintomas no nível de objeto
- [Balanceamento de acoplamento](../acoplamento/balanceamento.md)
