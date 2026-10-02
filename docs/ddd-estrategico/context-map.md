# Context map

Traçadas as fronteiras, falta decidir **como os contextos conversam**. O context map é o desenho dessas relações — e cada relação tem um padrão, com custos diferentes.

## Upstream e downstream

Numa relação entre dois contextos, o **upstream** é quem fornece e o **downstream** é quem consome. A seta de dependência aponta do downstream para o upstream; o *conhecimento* flui no sentido contrário. Quem está downstream é afetado pelas mudanças de quem está upstream — por isso o padrão de integração existe: ele decide **quanto** dessa mudança atravessa.

## Os padrões

| Padrão | Em uma frase | Quando usar |
|---|---|---|
| **Anti-Corruption Layer** | O downstream traduz o modelo do upstream para o seu, numa camada própria. | Consumir um modelo que você não controla: provider externo, sistema legado. |
| **Open Host Service** / **Published Language** | O upstream publica um contrato estável, separado do modelo interno. | Um contexto consumido por vários outros. |
| **Customer / Supplier** | O downstream negocia o que precisa e o upstream entrega. | Os dois lados estão sob o mesmo time ou times que colaboram. |
| **Conformist** | O downstream aceita o modelo do upstream como ele é. | Não há poder de negociação e o modelo do upstream é bom o bastante. |
| **Shared Kernel** | Dois contextos compartilham um pedaço de modelo. | Raramente — e com muita disciplina. |
| **Separate Ways** | Os contextos não se integram; cada um resolve do seu jeito. | Quando integrar custa mais do que duplicar. |

## Os dois extremos que mais aparecem

### ACL: o downstream se protege

A Anti-Corruption Layer é o padrão mais usado na fronteira com o mundo externo. Ela tem uma regra só, que é fácil de violar: **a borda traduz, não decide**. Essa regra merece uma nota própria — veja [Anti-Corruption Layer](../arquitetura-de-fronteiras/acl.md).

### Shared Kernel: o acoplamento que parece inofensivo

Compartilhar um pedaço de modelo parece economia. Na prática, todo shared kernel é **acoplamento funcional simétrico** em potencial: a mudança num lado obriga a mudança no outro, e ninguém sabe ao certo quem é o dono.

Shared kernel funciona quando o pedaço compartilhado é **pequeno e estável** — value objects como `LanguageCode` ou `FilePath`, que quase nunca mudam. Ele dói quando vira o atalho para não desenhar um contrato.

!!! example "Um caso real"
    No HomeFlix, três módulos leem value objects de configuração do domínio do módulo `settings`. Aqui o compartilhamento é **declarado**: um ADR aceita essa variante para configuração somente leitura, tratando os VOs como contrato publicado, e assume o trade-off por escrito. Mesmo assim, `media` e `settings` são o par que mais muda junto no histórico — e entender *por quê* é o que separa um shared kernel saudável de um problemático. A análise está no [estudo de caso](../estudos-de-caso/homeflix/auditoria-de-acoplamento.md).

## Separate Ways é uma escolha válida

Duplicar não é sempre pecado. Se dois contextos precisam de uma lógica parecida mas evoluem por motivos diferentes, duplicar conscientemente é melhor do que criar uma dependência que vai amarrar os dois. O teste é o mesmo de sempre: *esses dois pedaços mudam juntos?*

## Relacionados

- [Bounded contexts](bounded-contexts.md)
- [Anti-Corruption Layer](../arquitetura-de-fronteiras/acl.md)
- [Integration strength](../acoplamento/integration-strength.md)
