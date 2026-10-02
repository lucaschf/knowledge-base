# Base de Conhecimento

Notas de engenharia, arquitetura e design de software — DDD estratégico e tático, fronteiras entre contextos e acoplamento, com um sistema real ([HomeFlix](https://github.com/lucaschf/homeflix)) como estudo de caso. **A fonte da verdade é markdown** — o site é só uma forma de navegar.

📖 **Site:** [lucaschf.github.io/knowledge-base](https://lucaschf.github.io/knowledge-base/)

## Rodar localmente

```bash
uv sync                           # cria o .venv e instala as dependências
uv run mkdocs serve               # http://127.0.0.1:8000 com hot-reload
```

> Precisa do [uv](https://docs.astral.sh/uv/) instalado. Para adicionar dependências: `uv add <pacote>`.

## Publicar

O deploy é automático: todo push na `main` dispara o workflow `.github/workflows/deploy.yml`,
que faz `mkdocs build --strict` e publica no GitHub Pages.

Setup único no repositório (uma vez): **Settings → Pages → Source: GitHub Actions**.

## Estrutura

```
docs/
├── index.md                    # porta de entrada
├── modelagem-de-dominio/       # DDD tático: code smells
│   └── code-smells/
├── ddd-estrategico/            # subdomínios, bounded contexts, context map
├── arquitetura-de-fronteiras/  # ACL, read ports, eventos entre contextos
├── acoplamento/                # modelo de Khononov: strength, distance, volatility
└── estudos-de-caso/
    └── homeflix/               # mapa de contextos, auditoria, decisões
```

## Convenções

- **Markdown sempre.** HTML artesanal só quando interatividade agrega de verdade.
- **Diagramas em Mermaid**, dentro do markdown — versionável e regenerável.
- Uma nota = um conceito. Linke entre notas em vez de duplicar.
- Cada conceito segue o mesmo esqueleto: _o que é → como detectar → o que fazer → antes/depois_.
