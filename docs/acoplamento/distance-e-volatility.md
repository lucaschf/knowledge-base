# Distance e volatility

A [força de integração](integration-strength.md) diz *o que* está acoplado. Distance e volatility dizem **quanto isso vai custar**.

## Distance: onde o acoplamento mora

A distância é medida pelo **ancestral comum mais próximo** dos dois componentes. Quanto mais longe esse ancestral, mais cara é uma mudança coordenada.

| Ancestral comum | Distância | Exemplo |
|---|---|---|
| Mesmo método | Mínima | Duas linhas no mesmo método |
| Mesma classe | Muito baixa | Métodos do mesmo agregado |
| Mesmo módulo | Baixa | Use case e entidade do mesmo bounded context |
| Mesmo serviço, módulos irmãos | Média | Dois módulos num monólito modular |
| Serviços distintos | Alta | Chamada HTTP para outro serviço do sistema |
| Sistemas ou organizações diferentes | Máxima | API de um provider externo |

Duas perguntas que ajudam a estimar: *precisa de deploy coordenado?* e *se um cai, o outro para?*

### O fator social

Módulos mantidos por **times diferentes** estão mais distantes do que o código sugere: uma mudança coordenada agora exige uma conversa, uma reunião, um alinhamento de prioridade. Na prática, suba a distância em um nível quando a fronteira também é uma fronteira de time (é a Lei de Conway aplicada ao custo de mudança).

Mantenedor solo tem fator social perto de zero — mas vale registrar onde ele *passaria* a existir se o time crescesse, porque é ali que um acoplamento hoje barato fica caro.

## Volatility: com que frequência muda

Acoplamento com algo que nunca muda não cobra juros. A volatilidade é o multiplicador que decide se um acoplamento forte é problema ou não.

### Pelo tipo de subdomínio

A primeira estimativa vem da [classificação de subdomínios](../ddd-estrategico/subdominios.md):

| Tipo | Volatilidade esperada |
|---|---|
| **Core** | Alta — é onde o negócio quer evoluir |
| **Supporting** | Baixa |
| **Generic** | Mínima — *quando é comprado pronto* |

### Pelo histórico real

A estimativa pelo rótulo é um palpite. O git é a medição:

```bash
# Commits por módulo nos últimos 6 meses
git log --since="6 months ago" --format="%H" --no-merges | while read c; do
  git show --format="" --name-only "$c" | grep "^src/modules/" | cut -d/ -f3 | sort -u
done | sort | uniq -c | sort -rn
```

!!! warning "Quando o rótulo e o git discordam, o git ganha"
    No HomeFlix, o streaming é classificado como *Generic* — e foi o subdomínio com mais commits concentrados quando o catálogo foi auditado. A classificação não está errada: streaming é um problema resolvido por outros produtos. O que a heurística esquece é que **construir** o genérico do zero o torna volátil. Use o rótulo para entender o negócio; use o histórico para estimar o custo.

### Volatilidade herdada

Um módulo estável pode **herdar** volatilidade. Se um módulo supporting tem acoplamento functional ou intrusive com um módulo core, toda mudança no core vaza para ele — e ele passa a mudar com a frequência do core, mesmo sem ter motivo próprio.

### Sinais no código

Na falta de histórico: muitos `TODO` e `FIXME`, várias versões de API convivendo (`v1`, `v2`), testes frágeis que quebram a cada sprint e comentários do tipo "regra de negócio:" são sinais de área em movimento.

## Relacionados

- [Integration strength](integration-strength.md)
- [Balanceamento](balanceamento.md)
- [Subdomínios](../ddd-estrategico/subdominios.md)
