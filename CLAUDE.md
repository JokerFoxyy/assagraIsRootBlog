# CLAUDE.md

Guia para o Claude Code (e qualquer dev) trabalhar neste repositório. Leia
antes de criar páginas, componentes ou abrir PRs.

## O projeto

**AssagraIsRoot.dev** — blog técnico de engenharia de software de Victor
Assagra. Conteúdo focado em **Java, internals da JVM, arquitetura de software,
containers, Linux e segurança da informação**.

- **Domínio:** https://assagraisroot.dev
- **Repositório:** https://github.com/JokerFoxyy/assagraIsRootBlog
- **Deploy:** Vercel (build estático)

## Stack

| Camada | Tecnologia |
| --- | --- |
| Framework | Astro 4 (`output: 'static'`) |
| Conteúdo | Markdown / MDX via `@astrojs/mdx` + Content Collections |
| Linguagem | TypeScript (`strict`) |
| SEO | `@astrojs/sitemap` (gera `sitemap-index.xml`) |
| Hospedagem | Vercel |

> **Sem frameworks de UI extras.** Nada de React/Vue/Svelte a menos que seja
> estritamente necessário. Prefira componentes `.astro` nativos.

## Comandos

```bash
npm install        # instala dependências
npm run dev        # dev server em http://localhost:4321
npm run build      # astro check (type-check) + astro build -> dist/
npm run preview    # serve o build de produção localmente
```

O `build` roda `astro check` antes do `astro build`, então **um build limpo
garante zero erros de TypeScript**. É o portão de qualidade do projeto.

## Estrutura de pastas

```
src/
├── components/
│   └── PostCard.astro        # card de post reutilizável (home, /blog, /tags)
├── content/
│   ├── config.ts             # schema zod da collection `blog`
│   └── blog/                 # posts em .md / .mdx
├── layouts/
│   ├── Base.astro            # shell: <head>, nav, footer, SEO/OG
│   └── Post.astro            # layout de artigo (usa Base por dentro)
├── pages/
│   ├── index.astro           # home: hero + 5 posts recentes
│   ├── sobre.astro           # portfólio (stack, projetos, certs)
│   ├── blog/
│   │   ├── index.astro       # listagem completa + filtro de tags (client-side)
│   │   └── [...slug].astro   # rota dinâmica de cada post
│   └── tags/
│       └── [tag].astro       # rota estática por tag
└── styles/
    └── global.css            # design system (CSS variables + reset)
public/                       # assets servidos como estão (favicon, og-default)
```

## Design system — regras inegociáveis

Todas as cores e fontes vivem como **CSS variables** em `src/styles/global.css`
(`:root`). **Nunca use cores hardcoded** fora dessas variáveis.

```
--color-bg:          #0d1117   --color-text:        #e6edf3
--color-surface:     #161b22   --color-text-muted:  #8b949e
--color-border:      #30363d   --color-accent:      #58a6ff
                               --color-accent-warm: #f78166

--font-sans: 'IBM Plex Sans', system-ui, sans-serif   (corpo)
--font-mono: 'JetBrains Mono', monospace               (código / UI mono)
```

Tema dark inspirado no GitHub dark. Layout centralizado com
`--content-width: 780px`. Tudo deve ser **responsivo (mobile-first)**.

## Convenções

- **Páginas:** `kebab-case` (`sobre.astro`, `blog/index.astro`).
- **Componentes:** `PascalCase` (`PostCard.astro`).
- **Toda página usa o layout `Base.astro`** (direta ou indiretamente).
- **JS client-side:** dentro de `<script>` no próprio `.astro` (escopo de
  módulo). Sem libs de interatividade externas. Ex.: o filtro de tags em
  `blog/index.astro`.
- **Nav ativo:** `Base.astro` detecta a rota com `Astro.url.pathname` e marca o
  link atual. Ao adicionar uma rota top-level, inclua-a em `navLinks`.

## Content Collections

A collection `blog` é validada por zod em `src/content/config.ts`:

```ts
{
  title:       string                 // obrigatório
  description: string                 // obrigatório (usado em cards e OG)
  pubDate:     z.coerce.date()        // aceita "2024-02-01"
  tags:        string[] | undefined   // opcional
  draft:       boolean (default false)// drafts são filtrados das listagens
}
```

### Criar um post novo

1. Crie `src/content/blog/<slug>.md` (ou `.mdx`).
2. Preencha o frontmatter conforme o schema acima.
3. O slug do arquivo vira a URL: `/blog/<slug>`.
4. Posts com `draft: true` não aparecem em nenhuma listagem nem geram rota.
5. Cada tag nova gera automaticamente uma página em `/tags/<tag>`.

Todas as queries filtram drafts com `getCollection('blog', ({ data }) => !data.draft)`.

## SEO / Open Graph

`Base.astro` aceita props opcionais: `ogTitle`, `ogDescription`, `ogImage`
(fallback `/og-default.svg`). `Post.astro` repassa título/descrição do artigo.
Canonical, `og:*` e `twitter:card` são resolvidos contra `site` do
`astro.config.mjs`.

## CI/CD (GitHub Actions)

| Workflow | Gatilho | O que faz |
| --- | --- | --- |
| `ci.yml` | push / PR | `npm ci` + `npm run build` (type-check + build) |
| `deploy.yml` | push em `main` / PR | deploy Vercel: produção em main, preview em PR |
| `auto-pr.yml` | push em `feature-*` | abre/atualiza PR automático para `main` |

### Fluxo de trabalho esperado

> **`main` é protegida — push direto é bloqueado.** Todo trabalho entra por
> PR a partir de um branch `feature-*`.

1. Crie um branch `feature-<algo>` a partir de `main`.
2. Faça commits e `git push`.
3. O `auto-pr.yml` abre um PR para `main` automaticamente (pushes seguintes só
   o atualizam — sem duplicar).
4. `ci.yml` valida o build; `deploy.yml` publica um preview e comenta a URL.
5. Após merge em `main`, `deploy.yml` publica em produção (`--prod`).

### Secrets necessários (Settings → Secrets → Actions)

`VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` — obtenha rodando
`vercel link` localmente (lê `.vercel/project.json`) ou no painel da Vercel.

> Se preferir usar **apenas** a integração nativa Vercel↔GitHub (que já
> publica sozinha), remova/desative `deploy.yml` para não duplicar deploys.

## Checklist antes de abrir PR

- [ ] `npm run build` passa limpo (sem erros de TypeScript).
- [ ] Cores/fontes vêm das CSS variables — nada hardcoded.
- [ ] Páginas novas usam `Base.astro` e estão no nav, se top-level.
- [ ] Interatividade é `<script>` nativo, sem libs extras.
- [ ] Componentes em PascalCase, páginas em kebab-case.
