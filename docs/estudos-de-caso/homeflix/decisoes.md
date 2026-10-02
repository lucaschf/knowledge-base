# Decisões que valem a leitura

O HomeFlix tem 36 ADRs. Estes seis são os que mais ensinam — não porque acertaram tudo, mas porque mostram o raciocínio, incluindo o que foi recusado e o que ficou para depois.

## ADR-007 — Entidades imutáveis com `with_*`

**O problema:** as entidades tinham métodos como `movie.add_genre("Sci-Fi")`, que alteravam o estado e não devolviam nada. Era fácil esquecer que a operação mutava, e mais fácil ainda escrever código que dependia disso sem perceber.

**A decisão:** toda entidade é `frozen`. Mudar o estado é criar uma nova instância — `movie = movie.with_genre(...)` —, e o `updated_at` é atualizado automaticamente em `with_updates()`, para ninguém precisar lembrar.

**Por que vale ler:** é a forma mais barata de eliminar uma classe inteira de bugs. Com imutabilidade, a pergunta "quem mais tem uma referência para este objeto e vai ver a mudança?" deixa de existir.

[Ler o ADR-007](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-007-immutable-entities-with-convention.md)

## ADR-009 — Read ports entre contextos

**O problema:** contextos lendo uns dos outros por import direto, cada um conhecendo as entidades do vizinho.

**A decisão:** toda leitura entre contextos passa por uma porta do consumidor, com tipos do consumidor, implementada por um adapter que é o único ponto de import do provedor. Uma variante mais leve é aceita para configuração somente leitura.

**Por que vale ler:** a variante. Em vez de aplicar a regra cegamente, o ADR reconhece onde ela vira burocracia (um DTO e um adapter por bucket de configuração) e aceita um alinhamento *parcial*, com o trade-off escrito. Detalhes em [Read ports entre contextos](../../arquitetura-de-fronteiras/read-ports.md).

[Ler o ADR-009](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-009-cross-bc-read-ports.md)

## ADR-018 — Identificadores como value objects nas fronteiras

**O problema:** a lista de bibliotecas que um perfil pode ver era uma lista de `str` da entidade até o banco. Um id com erro de digitação nunca casava com biblioteca nenhuma — e o perfil perdia o acesso em silêncio, de um jeito indistinguível de "acesso removido de propósito".

**A decisão:** ids viajam como value objects em toda fronteira, com `str` permitido só no HTTP e na coluna do banco. A conversão acontece uma vez, na primeira fronteira.

**Por que vale ler:** o nome que o ADR dá ao dano — *default-deny silencioso* — é um dos melhores argumentos contra [Primitive Obsession](../../modelagem-de-dominio/code-smells/primitive-obsession.md) que eu conheço. E o ADR é honesto sobre o custo: tipar todos os `media_id` tocaria cerca de 29 arquivos em 4 contextos, então essa parte ficou para uma migração incremental.

[Ler o ADR-018](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-018-domain-identifiers-as-vos-at-boundaries.md)

## ADR-020 — A porta no nível errado de abstração

**O problema:** a detecção de abertura era só por áudio, e falhava quando a música da abertura variava entre episódios. Um algoritmo por imagem (hash perceptual de frames) funcionava melhor — mas não dava para plugá-lo. A porta do detector recebia *fingerprints de áudio prontos*, e a extração de áudio morava no job, fora da porta.

**A decisão:** subir a porta para o nível "arquivos de episódio → marcadores". Cada adapter passa a ser dono do pipeline inteiro (extração, hash, correlação), e o algoritmo vira escolha de configuração em runtime.

**Por que vale ler:** é o exemplo mais claro que eu tenho de que **port/adapter não garante troca de implementação**. Se a porta recebe um tipo que só uma implementação produz, a abstração é de fachada. A pergunta certa ao desenhar uma porta é: *o que o caller tem nas mãos, e o que ele quer de volta?* — não *o que a implementação atual precisa*.

[Ler o ADR-020](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-020-pluggable-intro-detector-frame-hash.md)

## ADR-025 — Uma recomendação de auditoria recusada

**O problema:** uma auditoria de code smells apontou *Anemic Domain Model* na reconciliação de metadados — a regra de "o dado do TMDB sobrescreve ou só preenche o que está vazio" vivia nos use cases, e não nas entidades. A recomendação era mover para o domínio.

**A decisão:** não mover. A reconciliação lê DTOs do provider, que são da camada de aplicação. Colocá-la na entidade faria o domínio importar a aplicação — invertendo a regra de dependência. O ADR quebra a "regra de reconciliação" em quatro responsabilidades e mostra que só uma delas é de fato de domínio — e essa já estava lá.

**Por que vale ler:** porque **recusa uma recomendação com argumento**, registra que ela foi recusada duas vezes antes, e deixa escrito o gatilho para revisitar. Smell é pergunta, não veredito — e um ADR é o lugar certo para a resposta, para que o mesmo achado não volte em toda auditoria.

[Ler o ADR-025](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-025-provider-metadata-reconciliation-in-application.md)

## ADR-032 — Decompor o catálogo em subdomínios

**O problema:** o módulo de catálogo tinha cerca de 40 mil linhas e soldava subdomínios com volatilidades diferentes. Streaming, metadados e marcadores de reprodução mudavam por motivos que não tinham nada a ver com filmes e séries.

**A decisão:** extrair um subdomínio por vez (Strangler Fig), do menos acoplado para o mais acoplado: primeiro streaming, depois metadados, depois marcadores. Cada extração com seu PR e precedida da segregação do repositório correspondente.

**Por que vale ler:** pela ordem e pela cautela. O ADR descarta explicitamente a alternativa de fatiar por camada técnica, e lista como risco "módulos que acabam anêmicos, só transporte" — com a mitigação de só extrair o que tem volatilidade e linguagem próprias. O estado atual dessa migração aparece na [auditoria de acoplamento](auditoria-de-acoplamento.md).

[Ler o ADR-032](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-032-decompose-media-into-subdomains.md)

## Relacionados

- [Mapa de contextos](mapa-de-contextos.md)
- [Auditoria de acoplamento](auditoria-de-acoplamento.md)
- Índice completo: [docs/adr no HomeFlix](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/README.md)
