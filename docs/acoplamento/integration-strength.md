# Integration strength

A força de integração mede **quanto conhecimento** um componente precisa ter do outro para funcionar. Quanto mais conhecimento compartilhado, mais motivos para uma mudança de um lado quebrar o outro.

São quatro níveis, do mais forte ao mais fraco.

## 1. Intrusive — o mais forte

O consumidor depende de detalhes internos que o provedor **nunca desenhou para integração**.

**Sinais:**

- Ler a tabela ou a coleção de outro módulo direto do banco.
- Importar um membro privado (`_algo`) de outro módulo.
- Depender da estrutura interna de configuração alheia; monkey-patching.

**Por que é o pior:** o provedor não sabe que está sendo usado assim. Qualquer refatoração interna, por mais inocente, quebra o consumidor sem aviso.

```python
# Intrusive: um módulo usando um helper privado da infraestrutura de outro
from src.modules.streaming.infrastructure.streaming._subprocess import with_ffmpeg_threads
```

Esse import existiu no HomeFlix até outubro de 2026. O [estudo de caso](../estudos-de-caso/homeflix/auditoria-de-acoplamento.md) mostra como foi encontrado e removido.

## 2. Functional

Os componentes implementam funcionalidades **inter-relacionadas**: um não está correto sem o outro. Três graus:

| Grau | O que é | Sinal |
|---|---|---|
| **Sequential** | Ordem obrigatória de chamada entre módulos. | "Chame `prepare()` antes de `run()`" atravessando fronteiras. |
| **Transactional** | Tudo ou nada entre módulos. | Sagas, compensações, deploy casado. |
| **Symmetric** | A mesma regra de negócio implementada em dois lugares, que precisam andar juntos. | Validação duplicada; comentário "lembrar de atualizar X quando mudar Y". |

O functional symmetric é o mais traiçoeiro: nenhuma ferramenta estática o encontra, porque não há import nenhum — só duas cópias da mesma regra que divergem em silêncio.

## 3. Model

O provedor expõe **seu modelo de domínio** na interface pública. O consumidor recebe a entidade inteira quando precisava de um campo.

**Sinais:**

- Uma entidade de domínio retornada para outro módulo.
- Um enum interno compartilhado com outros contextos.
- Um módulo importando value objects do domínio de outro.

Dentro do model coupling há graus, medidos por *connascence*: concordar só no **nome** de um campo é mais fraco do que concordar no **tipo**, que é mais fraco do que concordar no **significado** (o que `status = 3` quer dizer) ou no **algoritmo** (os dois lados calculam o hash do mesmo jeito).

## 4. Contract — o mais fraco

O provedor expõe um **modelo de integração dedicado**, separado do interno: um DTO, um snapshot, uma linguagem publicada. O modelo interno pode mudar à vontade enquanto o contrato for respeitado.

**É o que fazem:** Facade, Adapter, Anti-Corruption Layer, [read ports](../arquitetura-de-fronteiras/read-ports.md) com tipos próprios, eventos de integração bem desenhados.

## Referência rápida

| Padrão encontrado | Força | Ação |
|---|---|---|
| Ler o banco de outro módulo | Intrusive | Refatorar com urgência |
| Importar `_privado` de outro módulo | Intrusive | Refatorar com urgência |
| Regra de negócio duplicada em dois módulos | Functional (symmetric) | Dar um dono só para a regra |
| Saga ou transação distribuída | Functional (transactional) | Avaliar se juntar os módulos não é melhor |
| Ordem obrigatória de chamada entre módulos | Functional (sequential) | Encapsular o protocolo |
| Entidade de domínio retornada para outro módulo | Model | Criar um DTO de integração |
| Enum interno compartilhado | Model | Criar um enum de contrato público |
| DTO ou snapshot dedicado por caso de uso | Contract | ✅ |
| Porta com tipos próprios + adapter | Contract | ✅ |

## A pergunta que resume

> *Se eu mudar um detalhe interno deste componente, quantos outros quebram — e eles sabiam que dependiam disso?*

A segunda metade da pergunta é o que separa intrusive de model: no model coupling, pelo menos, a dependência é visível na interface.

## Relacionados

- [Distance e volatility](distance-e-volatility.md)
- [Balanceamento](balanceamento.md)
- [Read ports entre contextos](../arquitetura-de-fronteiras/read-ports.md) — um padrão que aceita model coupling de forma confinada
