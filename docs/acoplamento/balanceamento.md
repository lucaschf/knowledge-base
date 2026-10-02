# Balanceamento

Com as três dimensões medidas, o diagnóstico de cada dependência sai de uma tabela. Esta nota é o roteiro para aplicar o modelo num código real.

## A tabela de diagnóstico

Para cada par acoplado, use uma escala simples (0 = baixo, 1 = alto):

| Dimensão | 0 | 1 |
|---|---|---|
| Strength | Contract | Intrusive |
| Distance | Mesmo módulo | Serviços distintos |
| Volatility | Generic / Supporting | Core |

```
esforço de manutenção = strength × distance × volatility
```

Qualquer dimensão em zero derruba o esforço. A tabela completa:

| Strength | Distance | Volatility | Diagnóstico |
|---|---|---|---|
| Alta | Alta | Alta | 🔴 **Crítico** — complexidade global e mudança cara |
| Alta | Alta | Baixa | 🟡 **Aceitável** — forte, mas estável (integração legada congelada) |
| Alta | Baixa | Alta | 🟢 **Bom** — coesão: muda junto, mora junto |
| Alta | Baixa | Baixa | 🟢 **Bom** — forte, mas parado |
| Baixa | Alta | Alta | 🟢 **Bom** — baixo acoplamento |
| Baixa | Alta | Baixa | 🟢 **Bom** — baixo acoplamento e estável |
| Baixa | Baixa | Alta | 🟠 **Atenção** — complexidade local: coisas não relacionadas no mesmo lugar |
| Baixa | Baixa | Baixa | 🟡 **Aceitável** — pode gerar ruído, custo baixo |

O caso 🟠 costuma ser esquecido: dois componentes que **não** se conhecem, mas moram no mesmo módulo e mudam muito, tornam o módulo difícil de entender. É o [Divergent Change](../modelagem-de-dominio/code-smells/divergent-change.md) no nível do módulo.

## O roteiro

### 1. Contexto

Defina o escopo (o sistema todo ou uma área) e classifique cada módulo em [Core, Supporting ou Generic](../ddd-estrategico/subdominios.md). Essa classificação é o primeiro palpite de volatilidade.

### 2. Mapa de dependências

Liste os imports entre módulos. Num monólito modular em Python:

```bash
grep -rn "^from src\.modules\." src/modules --include="*.py" | grep -v test
```

Monte o grafo: nós são módulos, arestas são dependências. Lembre que se A depende de B, B está *upstream* — o conhecimento flui de B para A.

### 3. Classifique cada aresta

Para cada dependência, a [força](integration-strength.md) (intrusive, functional, model, contract) e a [distância](distance-e-volatility.md).

### 4. Meça a volatilidade de verdade

Conte commits por módulo e, principalmente, **pares que mudam juntos**:

```bash
git log --since="6 months ago" --format="%H" --no-merges | while read c; do
  git show --format="" --name-only "$c" | grep "^src/modules/" | cut -d/ -f3 | sort -u | paste -sd "+" -
done | grep "+" | sort | uniq -c | sort -rn | head -15
```

Co-change alto entre dois módulos sem dependência declarada é sinal de **functional coupling** que ninguém escreveu.

### 5. Aplique a tabela e calibre

Todo par 🔴 ou 🟠 passa por três perguntas antes de virar recomendação:

1. **Causa dano concreto?** Deploy casado, bug em cascata, mudança que exige tocar N módulos. Sem dano que dê para nomear, é observação.
2. **A correção compensa?** Reencapsular tem custo. Acoplamento forte numa área parada há dois anos raramente paga a refatoração.
3. **O movimento é claro?** Diga qual dimensão corrigir: reduzir a força (criar um contrato), reduzir a distância (juntar o que muda junto) ou aceitar (registrar como dívida consciente).

## Os três movimentos possíveis

| Movimento | Quando | Como |
|---|---|---|
| **Reduzir strength** | O par está distante e precisa continuar distante. | Criar contrato: DTO, porta, evento de integração. |
| **Reduzir distance** | O par muda junto o tempo todo. | Juntar os componentes no mesmo módulo. |
| **Aceitar** | O par é estável ou a correção custa mais do que o problema. | Registrar a dívida, de preferência num ADR. |

!!! tip "Juntar também é resposta"
    A recomendação reflexa é sempre separar. Mas se dois módulos mudam juntos em quase todo commit, a fronteira entre eles está no lugar errado — e juntá-los elimina um acoplamento caro sem criar contrato nenhum. O critério é o co-change: alto, junte; baixo, separe.

## Regra de projeto não é o mesmo que equilíbrio

Um ponto que a prática ensina: **violar uma regra de arquitetura do projeto nem sempre é um desequilíbrio**, e vice-versa. Um import proibido de um contrato estável e pouco volátil pode estar perfeitamente equilibrado — e um uso permitido pode ser o par mais caro do sistema. A regra existe para prevenir; o modelo existe para medir. Use os dois.

O [estudo de caso do HomeFlix](../estudos-de-caso/homeflix/auditoria-de-acoplamento.md) aplica este roteiro de ponta a ponta e mostra esse descompasso em dados reais.

## Relacionados

- [Integration strength](integration-strength.md)
- [Distance e volatility](distance-e-volatility.md)
- [Coesão e problemas de fronteira](../ddd-estrategico/coesao-e-fronteiras.md)
