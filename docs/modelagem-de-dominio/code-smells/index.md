# Code Smells — DDD / Clean Architecture

Catálogo de **cheiros de modelagem de domínio e design de objetos**. Não é um linter de estilo — foca nos smells que separam "uso DDD" de "modelo o domínio".

## Princípio diretor: smell é pergunta, não veredito

Um code smell **sinaliza onde investigar**, não obriga refatorar. Marcar um `int` de contador de loop como Primitive Obsession é tão errado quanto não ver o `email: str`. O valor está em **achar os que importam e ignorar os que não importam**.

## O peso depende da camada

O mesmo cheiro é grave no domínio e tolerável na borda.

| Camada | Tolerância |
|---|---|
| **Domain** (entities, VOs, aggregates, domain services) | Rigor máximo. Anemia, Primitive Obsession e Stringly-Typed em conceito de domínio = severidade alta por padrão. |
| **Application** (use cases) | Feature Envy, Boolean Blindness e Temporal Coupling aparecem aqui. Lógica de _negócio_ no use case que deveria estar na entity = anemia. |
| **Infrastructure / Presentation** | Tolerância maior. Um DTO de request com vários campos primitivos é normal — a invariante vira VO ao _cruzar_ pro domínio, não antes. |

## Os 9 smells, por princípio violado

=== "Estado ilegal representável"

    A invariante foi empurrada pro runtime em vez de codificada no tipo.

    | Smell | Em uma frase |
    |---|---|
    | [Primitive Obsession](primitive-obsession.md) | Primitivos (`str`, `int`) onde existe um conceito de domínio com regras próprias. |
    | [Stringly-Typed Code](stringly-typed.md) | `str` carregando um conjunto fechado de valores (status, tipo) que deveria ser enum/tipo. |
    | [Boolean Blindness](boolean-blindness.md) | `bool` (ou flag arg) que não diz o que `True` significa no ponto de chamada. |
    | [Temporal Coupling](temporal-coupling.md) | Métodos que só funcionam se chamados numa ordem específica, sem o tipo impor isso. |

=== "Comportamento longe dos dados"

    _Tell, Don't Ask_ — o comportamento mora longe dos dados que ele usa.

    | Smell | Em uma frase |
    |---|---|
    | [Anemic Domain Model](anemic-domain-model.md) | Entities só com getters/setters; a regra de negócio vive em services de fora. |
    | [Feature Envy](feature-envy.md) | Um método se interessa mais pelos dados de outro objeto do que pelos seus. |

=== "Coesão / acoplamento"

    Coisas relacionadas separadas, ou não-relacionadas juntas.

    | Smell | Em uma frase |
    |---|---|
    | [Data Clumps](data-clumps.md) | Mesmo grupo de parâmetros andando junto por todo lado — pede um VO. |
    | [Shotgun Surgery](shotgun-surgery.md) | Uma mudança obriga editar muitos arquivos espalhados. |
    | [Divergent Change](divergent-change.md) | Um arquivo muda por muitos motivos diferentes (viola SRP). |

## Calibração antes de marcar

1. **Qual camada?** Borda tolera mais que domínio.
2. **Tem invariante de verdade?** Um primitivo sem regra própria não é Primitive Obsession.
3. **O custo da indireção compensa?** Extrair VO pra cálculo único pode não valer.
4. **É do PR ou dívida pré-existente?** Não bloqueie um PR por smell do entorno — registre à parte.
5. **Quem escreveu o código discorda?** Smell é heurística. Registre a divergência e siga.

## Relação com o resto da base

Este catálogo cobre o nível de objeto. Cheiros **estruturais** — import cruzado entre módulos, direção de dependência, fronteira de bounded context — estão em [Acoplamento](../../acoplamento/index.md) e [DDD Estratégico](../../ddd-estrategico/index.md).
