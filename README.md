# AssagraIsRoot.dev

Blog técnico de engenharia de software de **Victor Assagra** — Java, internals
da JVM, arquitetura, containers, Linux e segurança da informação.

🔗 https://assagraisroot.dev

## Stack

Astro 4 · MDX · TypeScript (strict) · Vercel · tema dark inspirado no GitHub.

## Desenvolvimento

```bash
npm install
npm run dev        # http://localhost:4321
npm run build      # type-check + build estático em dist/
npm run preview    # serve o build de produção
```

## Escrever um post

Crie `src/content/blog/<slug>.md` com o frontmatter:

```md
---
title: 'Título do artigo'
description: 'Resumo curto (aparece nos cards e no Open Graph).'
pubDate: 2024-02-01
tags: ['java', 'jvm']
draft: false
---
```

O arquivo vira a rota `/blog/<slug>`. Cada tag gera `/tags/<tag>`
automaticamente. Posts com `draft: true` ficam fora das listagens.

## Mais detalhes

Convenções, design system e fluxo de CI/CD estão documentados em
[`CLAUDE.md`](./CLAUDE.md).
