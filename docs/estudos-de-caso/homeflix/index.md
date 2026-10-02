# Estudo de caso: HomeFlix

O [HomeFlix](https://github.com/lucaschf/homeflix) é um servidor de streaming doméstico que eu mantenho como laboratório de Clean Architecture e DDD — e também como ferramenta de verdade, usada todo dia em casa. Quando os dois objetivos conflitam, a arquitetura ganha.

Ele serve de exemplo para o resto desta base porque é **código público e real**: as decisões têm ADR, o histórico está no git e dá para medir em vez de opinar.

## O sistema em números

- Monólito modular em Python (FastAPI, SQLAlchemy 2, Pydantic v2) com frontend React separado
- 11 bounded contexts em `src/modules/`
- 36 ADRs registrando as decisões de arquitetura
- 150+ endpoints REST e 4.400+ testes
- Cerca de 800 commits desde janeiro de 2026

## O que este estudo cobre

| Nota | O que mostra |
|---|---|
| [Mapa de contextos](mapa-de-contextos.md) | Os 11 contextos, a classificação de cada subdomínio e quem depende de quem — com volatilidade medida no git. |
| [Auditoria de acoplamento](auditoria-de-acoplamento.md) | O [roteiro de balanceamento](../../acoplamento/balanceamento.md) aplicado: 20 imports que contornam a regra de fronteira, classificados pelo modelo de Khononov. |
| [Decisões que valem a leitura](decisoes.md) | Seis ADRs e o raciocínio por trás de cada um, incluindo uma recomendação de auditoria que foi recusada — e por quê. |

!!! note "Um estudo honesto"
    Este não é um caso de sucesso polido. Ele mostra o que está bem resolvido e o que ainda não está: uma regra de fronteira sem trava no CI, uma migração em andamento, trade-offs aceitos de propósito. É o estado real de um sistema mantido por uma pessoa, em outubro de 2026.

## Links

- Código: [lucaschf/homeflix](https://github.com/lucaschf/homeflix) (backend) e [lucaschf/homeflix-web](https://github.com/lucaschf/homeflix-web) (frontend)
- Documentação do projeto e ADRs: [lucaschf.github.io/homeflix](https://lucaschf.github.io/homeflix/)
