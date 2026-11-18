# Auditando chamadas externas com um span

Quando uma chamada a um serviço externo precisa deixar trilha (o que foi enviado, o que voltou, quanto demorou, como terminou), a tentação é escrever um decorator por gateway com o próprio `try/except`. Funciona no primeiro. No terceiro, você tem três cópias quase iguais, e uma delas já diverge das outras justamente no caso de erro, que é o que a auditoria existe para registrar.

O padrão que resolve isso é separar **o que cada gateway sabe** do **que toda auditoria faz**, e colocar a parte comum num *span*: um objeto que envolve a chamada do começo ao fim, registra o desfecho e nunca muda esse desfecho.

## A divisão de responsabilidades

| Peça | Sabe sobre |
|---|---|
| **Decorator do port** | Como chamar o gateway, o que vai no registro como requisição (sem segredos), de onde vem a resposta bruta |
| **Span** | Latência, classificação do erro, montagem e gravação do registro, o que fazer quando a gravação falha |
| **Perfil do gateway** | Dados estáticos: tipo do provider, sensibilidade dos dados, quais erros do módulo significam falha de autenticação |

O decorator fica fino. Um gateway de cotação de frete, por exemplo:

```python
class TrackedShippingQuotes(ShippingQuotes):
    PROFILE = GatewayProfile(
        kind="shipping_quote",
        sensitivity=Sensitivity.PERSONAL,  # o CEP do cliente
        auth_errors=(CarrierAuthError,),
    )

    def quote(self, zip_code: ZipCode, package: Package) -> ShippingQuote:
        request = {"zip_code": zip_code.value, "weight_g": package.weight_g}
        with self._audit.track(self.PROFILE, gateway=self.name, request=request) as span:
            quote = self._inner.quote(zip_code, package)
            span.record_response(quote.raw)
            return quote
```

Não há `try/except` no decorator, nem acesso ao repositório de auditoria. Um gateway novo é um perfil e um bloco `with`.

## O span é um context manager que só observa

É o mesmo modelo do `start_as_current_span` do OpenTelemetry: o bloco `with` delimita a operação, e o `__exit__` registra como ela terminou. A diferença em relação à telemetria é que aqui o registro é a auditoria, mas a regra de ouro é a mesma: **a instrumentação nunca derruba a operação que observa.**

Para isso, o `__exit__` precisa respeitar o protocolo do `with` à risca:

| Como o bloco terminou | O que o `__exit__` recebe | O que o retorno decide |
|---|---|---|
| Normalmente, inclusive com `return` | `None, None, None` | Nada: o valor é ignorado |
| Com exceção | Tipo, instância e traceback | Verdadeiro **suprime** a exceção; falso faz o Python **relançar a original** |
| O próprio `__exit__` levantou | — | A nova exceção **substitui** a original |

Daí saem as duas regras do span:

```python
def __exit__(self, exc_type, exc, tb) -> Literal[False]:
    try:
        self._trail.add(self._closing_entry(exc))
    except Exception as failure:
        self._report(failure)  # never raise from __exit__
    return False  # Python re-raises the original exception, if any
```

1. **Sempre devolver `False`.** Quem relança o erro do gateway é o interpretador, não o seu código. A anotação `Literal[False]` (em vez de `bool`) diz ao mypy e ao pyright que o context manager nunca suprime exceções, e isso melhora a análise de alcançabilidade depois do `with`.
2. **Nunca deixar escapar a falha de gravação.** Pela terceira linha da tabela, um erro do banco de auditoria levantado dentro do `__exit__` tomaria o lugar do erro do gateway. Quem chamou receberia "banco fora do ar" em vez de "timeout na transportadora".

!!! warning "O bug clássico é um `raise` dentro do `finally`"
    A versão ingênua grava o registro num `finally` e relança a falha de gravação ali. É exatamente a terceira linha da tabela: o erro do gateway some.

O `__exit__` recebe qualquer `BaseException`, incluindo `SystemExit` e `KeyboardInterrupt`. Isso é uma vantagem sobre o `try/except Exception`: uma chamada abortada no meio (worker morto por timeout, por exemplo) também vira registro de falha, em vez de passar sem rastro ou, pior, ser gravada como sucesso.

## Classifique pela cadeia, não pela exceção de cima

Gateways HTTP raramente levantam o erro "real". Uma camada de retry esgota as tentativas e relança algo como `CarrierUnavailableError(...) from RetryError`, e o timeout verdadeiro fica dois níveis abaixo. Classificar só a exceção de cima grava todo timeout como erro genérico.

```python
def causes(error: BaseException | None) -> Iterator[BaseException]:
    """Yield the error and everything behind it, unwrapping retries."""
    visited: set[int] = set()
    while error is not None and id(error) not in visited:
        visited.add(id(error))
        yield error
        error = (
            error.last_attempt.exception()
            if isinstance(error, RetryError)
            else error.__cause__ or error.__context__
        )
```

A classificação percorre essa cadeia e para no primeiro elo que reconhece: autenticação, resposta ilegível, timeout. O resto é erro do provider. Os erros de autenticação específicos de cada módulo entram pelo perfil, para que a classificação não precise conhecer os módulos.

## Capture a resposta antes de mapear

Se o gateway tem duas fases (I/O e tradução para o modelo do domínio), entregue a resposta bruta ao span **entre** as duas:

```python
with self._audit.track(...) as span:
    raw = self._inner.fetch_raw(...)
    span.record_response(raw)
    return self._inner.map(raw)  # se falhar aqui, o raw já está no span
```

Assim, uma falha de mapeamento ainda deixa no registro a resposta que não pôde ser traduzida, que é justamente o que você vai querer ver ao investigar.

## Por que não uma classe base

O reflexo é uma `AuditedGatewayBase` com Template Method. Não funciona bem aqui:

- **Os ports têm assinaturas diferentes** (`quote(zip_code, package)`, `authorize(order)`, `track(code)`), então não existe um método-template público comum. A base acabaria expondo um `_audited_call(fn)` protegido, que é composição disfarçada de herança.
- **O que varia entre gateways é dado, não passo de algoritmo.** Tipo, sensibilidade e erros de autenticação cabem num perfil.
- **Seriam duas hierarquias.** O decorator já herda o port; uma base por cima acopla todos os gateways aos detalhes internos dela.

Uma API com várias funções (`run(fetch=..., response_of=..., translate=...)`) também resolve, mas esconde o fluxo: para entender o decorator, é preciso abrir o auditor e descobrir em que ordem cada função roda. O bloco `with` lê na ordem em que executa.

## Armadilhas

- **Lambda capturando a variável do `except`.** `except Exception as e: log(lambda: str(e))` só funciona se o lambda rodar dentro do bloco, porque o Python apaga `e` ao sair dele. Linters como o ruff acusam (F821). Passe o erro para um método.
- **Referência, não cópia.** Se o span guarda o dict passado a `record_response` e só o serializa no fim, um mapeamento que altere esse dict altera o registro. Se o mapeamento muda a entrada, passe uma cópia.
- **Segredos no registro.** A requisição auditada leva o que identifica a chamada, nunca chave de API, token ou header de autenticação.
- **Fallback silencioso.** Se a resposta não serializa e o registro sai só com metadados, logue um aviso (sem o payload, que pode ter dado pessoal). Senão você perde auditoria sem perceber.

## Relacionados

- [Auditoria obrigatória: falhe antes da chamada](auditoria-fail-closed.md) — quando a trilha de auditoria não pode faltar
- [Anti-Corruption Layer](acl.md) — o adapter que este decorator envolve
- [Temporal Coupling](../modelagem-de-dominio/code-smells/temporal-coupling.md) — o que o bloco `with` evita ao amarrar abertura e fechamento
