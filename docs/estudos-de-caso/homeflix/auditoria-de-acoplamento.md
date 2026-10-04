# Auditoria de acoplamento do HomeFlix

Esta nota aplica o [roteiro de balanceamento](../../acoplamento/balanceamento.md) ao HomeFlix, sobre o código de outubro de 2026.

## A regra do projeto

O HomeFlix tem uma regra de fronteira escrita no [ADR-009](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-009-cross-bc-read-ports.md): um módulo só lê outro por uma porta própria, e **o único arquivo que pode importar o outro módulo é o adapter** em `infrastructure/acl/`. Há duas exceções documentadas: o contrato publicado de presentation ([ADR-024](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-024-published-presentation-contracts-cross-bc.md)) e uma variante para configuração somente leitura, descrita no próprio ADR-009.

## O que a varredura encontrou

Procurando imports entre módulos fora de `infrastructure/acl/` e fora do contrato de presentation:

| De → para | Camada importada | Quantos | Situação |
|---|---|---|---|
| `media` → `metadata` | application (porta e DTOs) | 11 | Dívida de migração |
| `identity`, `media`, `streaming` → `settings` | domain (VOs de configuração) | 5 | Variante aceita no ADR-009 |
| ~~`media` → `streaming`~~ | ~~infrastructure (módulo privado `_subprocess`)~~ | ~~4~~ | Corrigido em outubro de 2026 |

Na primeira versão desta auditoria eram **20 imports** que a regra, lida ao pé da letra, não permite; hoje são 16. Mas o número sozinho engana: eles têm naturezas completamente diferentes, e é aí que o modelo de Khononov ajuda.

## Classificando cada grupo

### `media` → `streaming._subprocess`: intrusive (corrigido)

```python
from src.modules.streaming.infrastructure.streaming._subprocess import with_ffmpeg_threads
```

Quatro componentes de detecção do catálogo (extração e fingerprint de áudio, frame-hash, créditos) usam helpers de ffmpeg que moram num módulo **privado** — o underscore no nome diz que ele não foi feito para ser importado de fora.

| Strength | Distance | Volatility | Diagnóstico |
|---|---|---|---|
| Intrusive | Média (módulos irmãos) | Baixa | 🟡 Aceitável pela tabela |

A tabela diz "aceitável" porque o helper quase não muda. Mesmo assim, é o grupo que eu corrigiria primeiro: o custo é trivial e o import cria um **ciclo** (`streaming` lê de `media`, e `media` importa `streaming`). O que o catálogo usa dali não tem nada de streaming — são argumentos de subprocesso e o limite de threads do ffmpeg —, então o lugar natural para esse pedaço é `building_blocks/infrastructure`, de onde os dois módulos podem importar sem se conhecerem.

**Movimento:** reduzir a força para zero, movendo o helper para o código compartilhado. **Esforço:** trivial.

**O que foi feito:** os helpers genéricos foram para [`building_blocks/infrastructure/ffmpeg_subprocess.py`](https://github.com/lucaschf/homeflix/blob/develop/src/building_blocks/infrastructure/ffmpeg_subprocess.py), e o `_subprocess` do streaming ficou só com as constantes de aceleração por hardware, que só o streaming usa. O ciclo entre os dois módulos deixou de existir. Foi uma mudança de import, sem nenhuma linha de lógica alterada — o que confirma o diagnóstico: o acoplamento era de endereço, não de conhecimento.

### `media` → `metadata`: dívida de migração

Onze use cases do catálogo importam a porta `MetadataProvider` e os DTOs dela, que agora vivem no módulo `metadata`.

A história explica: até agosto, essa porta morava dentro de `media`. O [ADR-032](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-032-decompose-media-into-subdomains.md) extraiu a integração com o TMDB para um módulo próprio, e os imports mudaram de endereço sem mudar de forma. O próprio ADR prevê o passo seguinte — a escrita de volta no catálogo por um evento de integração ou uma porta estreita —, que ainda não foi feito.

| Strength | Distance | Volatility | Diagnóstico |
|---|---|---|---|
| Model / Contract | Média | Baixa (Supporting, 8 commits) | 🟢 Equilibrado |

Pelo modelo, esse acoplamento está equilibrado: os DTOs da porta foram desenhados para integração, o `metadata` muda pouco e o par só mudou junto duas vezes. O problema aqui é de **regra**, não de **equilíbrio** — a regra 1 do ADR-009 diz que os tipos da porta pertencem ao consumidor, e aqui pertencem ao provedor.

**Movimento:** duas saídas legítimas. Terminar a migração, com o catálogo definindo sua própria porta e um adapter traduzindo. Ou declarar o `metadata` como *Open Host Service* — ele publica a interface e quem quiser consome — e registrar isso num ADR. A segunda é honesta se o `metadata` só tem um consumidor por ora. **Esforço:** médio.

### `identity`, `media`, `streaming` → `settings`: variante aceita

```python
if TYPE_CHECKING:
    from src.modules.settings.domain.value_objects import ScanDedupConfig
```

As portas de configuração desses módulos são `Protocol`s que devolvem os VOs do domínio de `settings`. O ADR-009 aceita isso explicitamente: para configuração somente leitura e imutável, um DTO e um adapter por bucket seriam burocracia desproporcional, e os VOs são tratados como contrato publicado. O import fica sob `TYPE_CHECKING` — só existe para o verificador de tipos.

| Strength | Distance | Volatility | Diagnóstico |
|---|---|---|---|
| Model (declarado como contrato) | Média | ? | Depende da leitura do co-change |

E aqui aparece o dado que mais chamou atenção: **`media` e `settings` são o par que mais muda junto** — 13 commits em seis meses.

Lido sozinho, o número sugere que o "contrato estável" não é tão estável. Lendo os commits, a história é outra: quase todos são features novas que trouxeram **um ajuste novo** junto — espelhamento de artwork, OCR de legendas, detecção de abertura e créditos, transcodificação por GPU, deduplicação agendada. São mudanças **aditivas**: um bucket de configuração novo, não uma quebra dos que já existem.

!!! tip "Co-change alto não é sinônimo de contrato instável"
    O script de co-change aponta onde olhar; ele não diz o que você vai encontrar. Aqui, ler os 13 commits trocou a hipótese "contrato frágil" por "toda feature configurável nasce em dois módulos" — que é a consequência esperada de centralizar configuração de runtime num lugar só ([ADR-013](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-013-runtime-settings-db-backed.md)).

**Movimento:** aceitar, como o ADR já faz. Uma melhoria barata seria mover os VOs de configuração de `settings/domain` para um local explicitamente publicado — um `settings/public.py`, no mesmo espírito do ADR-024. Isso deixa claro no código, e não só no ADR, que eles são contrato; e libera o domínio de `settings` para evoluir sem medo de quebrar consumidores.

## A trava: import-linter no CI

A regra estava escrita e os desvios explicados, mas nada impedia o próximo. Desde outubro de 2026, o [import-linter](https://import-linter.readthedocs.io/) roda no pre-commit e no CI, e a decisão está registrada no [ADR-037](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-037-import-linter-boundary-contracts.md). São dois contratos no [`pyproject.toml`](https://github.com/lucaschf/homeflix/blob/develop/pyproject.toml):

- **Independência** entre os 11 módulos, com cada exceção citando o ADR que a autoriza: a ACL, o contrato publicado de presentation e a rota chamando o composition root.
- **Camadas** dentro de cada módulo: presentation, infrastructure, application, domain — cada uma só importa as de baixo.

Três detalhes da implementação valem mais do que o contrato em si.

**A variante aceita sai de graça.** Os imports de `settings` estão sob `TYPE_CHECKING`, e o import-linter tem uma opção para ignorar esse tipo de import (`exclude_type_checking_imports`). A exceção do ADR-009 não precisou de nenhuma linha no contrato.

**A dívida fica congelada, não perdoada.** Os imports de `media` para a porta do `metadata` entraram no `ignore_imports` um a um — 9 deles; os outros 2 estão sob `TYPE_CHECKING`. Um use case novo que importe a porta reprova o build. O conjunto só pode diminuir, e a decisão de modelagem (porta própria ou Open Host) continua pendente, mas sem crescer enquanto espera.

```toml
ignore_imports = [
    "src.modules.*.infrastructure.acl.** -> src.modules.**",
    "src.modules.*.presentation.** -> src.modules.identity.presentation.public",
    "src.modules.*.presentation.** -> src.config.containers",
    # Dívida congelada (ADR-037): listada um a um para só poder diminuir
    "src.modules.media.application.use_cases.enrich_movie_metadata -> src.modules.metadata.application.ports.metadata_provider_port",
    # ...
]
```

**A ferramenta falhava aberta, e ninguém veria.** O import-linter monta o grafo com o grimp, que ignora diretórios sem `__init__.py`. O HomeFlix tinha 18 deles, incluindo o módulo `preferences` inteiro. O contrato passaria verde sem nunca ter olhado aqueles arquivos. A correção foi criar os `__init__.py` e um teste de arquitetura que reprova se um diretório perder o seu, ou se um módulo novo ficar fora da lista dos contratos.

!!! warning "Uma trava que não reprova nada também precisa ser testada"
    Antes de confiar no contrato, injete uma violação de propósito e veja o build quebrar. Aqui, um import proibido entre módulos e outro entre camadas foram colocados à mão: os dois contratos reprovaram, e só então o PR foi aberto.

O contrato de camadas tem uma limitação assumida: presentation pode importar infrastructure, porque todos os módulos fazem a injeção de dependências por ali. Separar as duas exigiria mexer nos 11 módulos, e ficou fora do escopo do ADR.

## Outras observações

- **Um id cru na porta de progresso.** O [ADR-018](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-018-domain-identifiers-as-vos-at-boundaries.md) pede identificadores como value objects também nas fronteiras. A [`ProgressLookupPort`](https://github.com/lucaschf/homeflix/blob/develop/src/modules/collections/application/ports/progress_lookup_port.py) recebe `media_ids: Sequence[str]` — o que o próprio ADR adia de propósito, para uma migração incremental — e também `profile_id: str`, que não tem essa desculpa: `ProfileId` já mora no shared kernel. O adapter converte na entrada, então o risco é baixo, mas um id malformado só é pego dentro do adapter, e não na borda do consumidor.
- **O que está bem.** Os read ports de `collections` são exemplares: tipos mínimos, tradução explícita e um único arquivo por dependência. Os eventos de integração moram num vocabulário publicado (`shared_kernel/integration_events`), e o handler de progresso é idempotente — apagar o que já foi apagado não faz nada. E os desvios que existem estão, em sua maioria, *explicados* em ADR — o que é raro.

## Resumo

| Grupo | Imports | Natureza | Movimento | Esforço |
|---|---|---|---|---|
| `media` → `streaming._subprocess` | 4 → 0 | Intrusive, cria ciclo | ✅ Helper movido para `building_blocks` | Trivial |
| `media` → `metadata` | 11 | Regra violada, mas equilibrado | Congelado no contrato; falta terminar a migração do ADR-032 ou declarar Open Host | Médio |
| → `settings` | 5 | Variante aceita | Publicar os VOs explicitamente | Pequeno |
| (todos) | — | ~~Sem trava~~ | ✅ import-linter no CI ([ADR-037](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-037-import-linter-boundary-contracts.md)) | Pequeno |

A lição que o caso deixa: **contar violações de regra e medir desequilíbrio são exercícios diferentes.** O grupo com mais imports é o mais equilibrado; o grupo com menos imports é o único intrusive. Sem o modelo, a prioridade sairia invertida.

## Relacionados

- [Balanceamento](../../acoplamento/balanceamento.md)
- [Integration strength](../../acoplamento/integration-strength.md)
- [Read ports entre contextos](../../arquitetura-de-fronteiras/read-ports.md)
