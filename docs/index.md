# Base de Conhecimento

> Notas vivas de engenharia, arquitetura e design de software. Crescem conforme eu avanço.

Sou Lucas Cristovam, engenheiro backend focado em arquitetura de software — Python, Clean Architecture e Domain-Driven Design. Esta base é onde organizo o que aprendo aplicando essas ideias em sistemas reais. Ela é **markdown versionado em git**: o site é só uma forma confortável de navegar e buscar, e nada aqui é definitivo.

## Áreas

<div class="grid cards" markdown>

-   :material-cube-outline: **Modelagem de Domínio**

    ---

    Code smells que separam "uso DDD" de "modelo o domínio": Primitive Obsession, Anemic Domain Model, Feature Envy e mais seis.

    [:octicons-arrow-right-24: Ir para a seção](modelagem-de-dominio/index.md)

-   :material-map-outline: **DDD Estratégico**

    ---

    Subdomínios, bounded contexts e context map. Onde traçar as fronteiras antes de modelar dentro delas.

    [:octicons-arrow-right-24: Ir para a seção](ddd-estrategico/index.md)

-   :material-shield-half-full: **Arquitetura de Fronteiras**

    ---

    Anti-Corruption Layer, read ports e eventos entre contextos. A borda traduz, não decide.

    [:octicons-arrow-right-24: Ir para a seção](arquitetura-de-fronteiras/index.md)

-   :material-link-variant: **Acoplamento**

    ---

    O modelo de Khononov: strength, distance e volatility. Medir antes de "desacoplar".

    [:octicons-arrow-right-24: Ir para a seção](acoplamento/index.md)

-   :material-movie-open-outline: **Estudo de caso: HomeFlix**

    ---

    Tudo isso aplicado num sistema público e real: mapa de contextos, auditoria de acoplamento com dados do git e as decisões por trás.

    [:octicons-arrow-right-24: Ir para o estudo](estudos-de-caso/homeflix/index.md)

</div>

## Por onde começar

- **Vindo de DDD tático?** Comece por [Subdomínios](ddd-estrategico/subdominios.md) e depois [Bounded contexts](ddd-estrategico/bounded-contexts.md).
- **Integrando com APIs externas?** [Anti-Corruption Layer](arquitetura-de-fronteiras/acl.md) e [os 8 eixos de auditoria](arquitetura-de-fronteiras/acl-eixos.md).
- **Desconfiando que os módulos estão acoplados demais?** [Balanceamento](acoplamento/balanceamento.md), e veja o roteiro aplicado na [auditoria do HomeFlix](estudos-de-caso/homeflix/auditoria-de-acoplamento.md).

## Como esta base funciona

- **Uma nota = um conceito.** Notas pequenas que se linkam, em vez de notas gigantes.
- **Revisito sem dó.** Se aprendo algo que contradiz uma nota antiga, edito a nota — não crio uma "v2".
- **Exemplos reais quando possível.** O estudo de caso usa código público, com link para cada arquivo citado.
- **Markdown é a fonte.** Diagramas em Mermaid dentro do próprio markdown, para não apodrecerem.

!!! tip "Achou um erro?"
    Toda página tem um botão de edição que leva ao arquivo no GitHub. Correções e discordâncias são bem-vindas.
