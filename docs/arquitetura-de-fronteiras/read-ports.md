# Read ports entre contextos

Num monólito modular, ler dado de outro bounded context é a tentação mais barata do mundo: está no mesmo processo, basta um `import`. O read port é o padrão que torna essa leitura **explícita, estreita e fácil de trocar**.

## A estrutura

```
src/modules/<consumidor>/
├── application/
│   └── ports/
│       └── <x>_lookup_port.py    # interface + tipos, na língua do consumidor
└── infrastructure/
    └── acl/
        └── <x>_lookup_adapter.py # implementa a porta usando o provedor
```

As regras que fazem o padrão funcionar:

1. **A porta pertence ao consumidor.** Os tipos que ela devolve são do consumidor — nunca a entidade do outro contexto.
2. **O nome segue a linguagem do consumidor.** Não é `MovieRepository`; é `MediaLookupPort`, do ponto de vista de quem consulta.
3. **O adapter é o único ponto que importa o provedor.** O use case só conhece a porta.
4. **A ACL mora no consumidor.** O provedor não sabe quem o consome.
5. **Uma porta por direção.** Se A lê de B e B lê de A, são duas portas, uma em cada lado.
6. **O wiring fica na composition root.** O container injeta o adapter; o use case nunca o instancia.

## Um exemplo real

No HomeFlix, as listas do usuário mostram quanto de cada filme já foi assistido. O progresso não pertence ao contexto de *Collections* — pertence a *Watch Progress*. A porta, no consumidor:

```python
class ProgressLookupPort(ABC):
    """Batch lookup of watch-progress fractions, scoped to a profile."""

    @abstractmethod
    async def get_progress(
        self, media_ids: Sequence[str], *, profile_id: str
    ) -> dict[str, float]:
        """Return the watched fraction in [0, 1] per media id."""
```

Repare no que a porta **não** expõe: nenhuma entidade `WatchProgress`, nenhuma posição em segundos, nenhum timestamp. As listas só precisam de uma fração entre 0 e 1, e é isso que a porta promete.

O adapter, na infraestrutura de *Collections*:

```python
class ProgressLookupAdapter(ProgressLookupPort):
    """Resolve watched fractions via the Watch Progress Unit of Work."""

    async def get_progress(self, media_ids, *, profile_id):
        if not media_ids:
            return {}
        profile = ProfileId(profile_id)
        typed_ids = [WatchableMediaId(m) for m in media_ids]
        async with self._watch_progress_uow_factory() as uow:
            progress_map = await uow.progress.find_by_media_ids(typed_ids, profile)
        return {m: p.percentage / 100.0 for m, p in progress_map.items()}
```

A tradução acontece na última linha: o provedor fala em percentual (0–100), o consumidor quer fração (0–1). Se *Watch Progress* mudar a forma como guarda o progresso, só este arquivo muda.

Código completo: [porta](https://github.com/lucaschf/homeflix/blob/develop/src/modules/collections/application/ports/progress_lookup_port.py) e [adapter](https://github.com/lucaschf/homeflix/blob/develop/src/modules/collections/infrastructure/acl/progress_lookup_adapter.py).

## O trade-off que o padrão aceita

O adapter acima abre a Unit of Work do outro contexto e usa o repositório e os value objects dele. Em termos de [integration strength](../acoplamento/integration-strength.md), isso é **model coupling**: o adapter conhece o modelo interno do provedor.

É uma escolha consciente. A alternativa — o provedor publicar um contrato próprio (*Open Host Service*) — custa mais e só se paga quando há muitos consumidores. O read port aceita o model coupling, mas **confina num arquivo só**, no lado do consumidor. Quando o provedor muda, o estrago tem endereço conhecido.

!!! tip "Quando promover para contrato publicado"
    Se três ou quatro contextos têm adapters quase idênticos lendo o mesmo provedor, o model coupling está espalhado — e é hora de o provedor publicar uma interface de leitura própria.

## Uma exceção estreita: presentation

Algumas coisas não cabem numa porta de domínio. A resolução do perfil ativo a partir da requisição HTTP, por exemplo, é uma primitiva de autenticação que toda rota precisa. O HomeFlix trata isso com um **contrato publicado de presentation**: cada módulo pode expor um `presentation/public.py` com `__all__`, e só o que está ali pode ser importado por outro contexto ([ADR-024](https://github.com/lucaschf/homeflix/blob/develop/docs/adr/ADR-024-published-presentation-contracts-cross-bc.md)). É uma exceção documentada e estreita — não abre a porta para leituras de domínio.

## Regra sem trava vira sugestão

Um padrão de fronteira só se sustenta se alguma coisa impede a violação. Em Python, a ferramenta é o [import-linter](https://github.com/seddonym/import-linter): um contrato declarativo que falha o CI quando um módulo importa o que não devia. Sem isso, a regra "só o adapter importa o provedor" depende de cada pessoa lembrar dela em cada PR — e o [estudo de caso do HomeFlix](../estudos-de-caso/homeflix/auditoria-de-acoplamento.md) mostra o que acontece quando ninguém trava.

## Relacionados

- [Anti-Corruption Layer](acl.md)
- [Eventos entre contextos](eventos-entre-contextos.md)
- [Integration strength](../acoplamento/integration-strength.md)
