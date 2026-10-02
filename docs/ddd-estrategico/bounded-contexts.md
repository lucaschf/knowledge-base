# Bounded contexts

Um bounded context é a fronteira dentro da qual **cada termo do domínio tem um significado só**. A fronteira é linguística primeiro e técnica depois: pastas, módulos e serviços são a forma de *materializar* uma fronteira que já existe na linguagem.

## O sinal mais forte: o mesmo termo, significados diferentes

Quando a mesma palavra muda de significado conforme quem fala, você está olhando para duas fronteiras.

| Termo | Num contexto | Em outro |
|---|---|---|
| **Produto** | Catálogo: nome, descrição, fotos, categoria | Estoque: SKU, quantidade, depósito, lote |
| **Cliente** | Vendas: carrinho, endereço de entrega, histórico de pedidos | Suporte: chamados abertos, canal preferido, nível de atendimento |
| **Episódio** | Catálogo: título, sinopse, número, temporada | Reprodução: arquivo, faixas de áudio, marcadores de abertura |

O último exemplo é real. No HomeFlix, os marcadores de abertura e créditos começaram como colunas no agregado `Episode` do catálogo. O [ADR-032](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-032-decompose-media-into-subdomains.md) registra a decisão de tirá-los de lá: são dado de *reprodução*, com outra volatilidade e outro vocabulário, que só por acidente moravam junto com o título e a sinopse.

## Como achar a fronteira

1. **Liste os conceitos de negócio**, ignorando infraestrutura: entidades, use cases, portas.
2. **Agrupe por vocabulário.** `Episode`, `Season`, `Library` falam a língua do catálogo; `Session`, `Profile`, `Credential` falam a língua de identidade.
3. **Procure onde o significado muda.** É o sinal mais confiável que existe.
4. **Veja o que anda junto.** Conceitos que se referenciam e aparecem nos mesmos use cases tendem a pertencer ao mesmo contexto.

## Heurísticas rápidas

- **"Consigo explicar esse contexto em uma frase, sem *e também*?"** O *e também* denuncia dois contextos.
- **"As invariantes são as mesmas nos dois lugares?"** Invariantes diferentes significam conceitos diferentes — mesmo que o nome seja igual.
- **"Quem é o dono desse conceito?"** Se ninguém sabe responder, provavelmente são dois ou três conceitos com o mesmo nome.

## Anti-patterns

!!! failure "Modelo único global"
    Um `User` para reger todos os casos. Cada contexto precisa de um pedaço diferente, o modelo cresce para acomodar todos e vira o lugar onde toda mudança colide. A saída é aceitar **vários modelos** e traduzir nas fronteiras.

!!! failure "Contexto por camada técnica"
    "O contexto dos controllers", "o contexto dos repositórios". Fronteira é por capacidade de negócio, nunca por camada. A camada técnica é uma subdivisão *dentro* de cada contexto.

!!! failure "Big Ball of Mud"
    Tudo conectado com tudo, vocabulários misturados. Não acontece por decisão — acontece por ausência de fronteiras explícitas.

## Bounded context não é microserviço

Um bounded context é uma fronteira de **modelo**; um serviço é uma fronteira de **deploy**. Num monólito modular, cada módulo é um bounded context e todos sobem juntos. Separar o deploy é uma decisão posterior e independente — e só faz sentido quando a fronteira de modelo já está correta. Microserviço com fronteira errada é um Big Ball of Mud distribuído.

## Relacionados

- [Subdomínios](subdominios.md)
- [Context map](context-map.md)
- [Read ports entre contextos](../arquitetura-de-fronteiras/read-ports.md)
