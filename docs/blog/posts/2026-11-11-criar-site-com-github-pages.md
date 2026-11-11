---
date: 2026-11-11
categories:
  - DevOps
tags:
  - github-pages
  - zensical
  - mkdocs
  - git
  - ci-cd
---

# Como criar um site com GitHub Pages do zero (passo a passo)

Quer um blog, uma documentação ou uma página pessoal? O GitHub Pages resolve
isso de graça — mas tem duas armadilhas que derrubam quase todo mundo na
primeira hora. Neste post vai o passo a passo completo, do repositório vazio
ao site no ar, já com o que aprendi errando: o **build Jekyll fantasma** que
sobrescreve o site e o **Pages que exige repo público** dependendo do seu plano.

<!-- more -->

## Antes de começar: o que você vai precisar

| Item | Para quê |
|---|---|
| Uma conta no GitHub | hospeda o repo e serve o site |
| `git` | versionamento |
| `gh` (GitHub CLI) | criar repo, ativar Pages, acompanhar o deploy |
| Python 3.11+ e [uv](https://docs.astral.sh/uv/) | gerar o site localmente (sem instalar nada global) |

Instalando o `gh` no Arch:

```bash
sudo pacman -S github-cli
gh auth login
```

## Passo 1 — Criar o repositório

Existem dois modelos de URL no GitHub Pages:

| Modelo | Repo | URL |
|---|---|---|
| **Site de usuário** | `jailsonh3.github.io` | `https://jailsonh3.github.io/` |
| **Site de projeto** | `meu-projeto` | `https://jailsonh3.github.io/meu-projeto/` |

O site de usuário é a raiz do seu domínio no GitHub — só pode existir **um**
por conta. O site de projeto vive num subdiretório e é o mais comum.

```bash
gh repo create jailsonh3/jailsonh3.github.io --public \
  --description "Diário de um dev: bugs, soluções e curiosidades" \
  --homepage "https://jailsonh3.github.io"

git clone git@github.com:jailsonh3/jailsonh3.github.io.git ~/workspace/jailsonh3.github.io
cd ~/workspace/jailsonh3.github.io
git config user.name "Jailson Silva"
git config user.email "jailsonh3@gmail.com"
git symbolic-ref HEAD refs/heads/main
```

> `git config` **sem `--global`** de propósito: identidade só neste repo, sem
> mexer na sua máquina. Se o `git` reclamar de identidade ao commitar, é isso.

## Passo 2 — Escolher o gerador de site estático

| Opção | Prós | Contras |
|---|---|---|
| **[Zensical](https://zensical.org/)** | Drop-in do MkDocs, plugins nativos de blog/tags, sem Node, sem sudo | Ecossistema mais novo |
| [MkDocs + Material](https://squidfunk.github.io/mkdocs-material/) | O mais documentado do mundo | **Material entra em EOL em mai/2027** (sucessor é justamente o Zensical) |
| [Astro](https://astro.build/) | Moderno, ilhas de interatividade | Exige Node.js (e sudo no Arch) |
| [Hugo](https://gohugo.io/) | Muito rápido | Exige Go ou binário instalado |
| Jekyll | Nativo do Pages | Ruby, verboso, build lento |

Escolhi **Zensical** porque lê o mesmo `mkdocs.yml` (ou seja, dá pra migrar do
Material sem reescrever nada) e roda sem instalar nada:

```bash
uv tool run --from zensical zensical build      # constrói
uv tool run --from zensical zensical serve      # preview em :8000
```

## Passo 3 — Estrutura do projeto

```
.
├─ mkdocs.yml                     # configuração do site
├─ .github/workflows/deploy.yml   # CI: build + publish
├─ .nojekyll                      # (opcional, mas salva — ver armadilha 1)
├─ docs/
│  ├─ index.md                    # página inicial
│  ├─ sobre.md
│  ├─ tags.md
│  └─ blog/
│     ├─ index.md                 # listagem de posts
│     └─ posts/
│        └─ 2026-11-11-meu-post.md
└─ README.md
```

Convenção do blog: `docs/blog/posts/AAAA-MM-DD-slug.md` — a data no nome já
define a ordenação.

## Passo 4 — Configurar o `mkdocs.yml`

```yaml
site_name: Diário de um Dev
site_url: https://jailsonh3.github.io/
site_description: Bugs, soluções e curiosidades do dia a dia de quem programa.
site_author: Jailson Silva
copyright: "© 2026 Jailson Silva"

theme:
  variant: modern          # modern | classic (classic = cara do Material)
  language: pt-BR
  font:
    text: Inter            # qualquer fonte do Google Fonts
    code: JetBrains Mono
  palette:
    - media: "(prefers-color-scheme: light)"
      scheme: default      # default = claro | slate = escuro
      primary: indigo      # cor principal
      accent: indigo       # cor de destaque
      toggle:
        icon: lucide/moon
        name: Modo escuro
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: lucide/sun
        name: Modo claro
  features:
    - navigation.instant
    - navigation.top
    - toc.follow
    - content.code.copy
    - search.suggest

plugins:
  - search
  - blog:
      blog_dir: blog
  - tags

markdown_extensions:
  - admonition
  - attr_list
  - tables
  - toc:
      permalink: true
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.superfences

nav:
  - Início: index.md
  - Blog: blog/index.md
  - Tags: tags.md
  - Sobre: sobre.md
```

## Passo 5 — O workflow de deploy

`.github/workflows/deploy.yml`:

```yaml
name: Deploy do blog para o GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - name: Instalar Zensical
        run: |
          python -m pip install --upgrade pip
          pip install zensical
      - name: Build
        run: zensical build --strict
      - uses: actions/upload-pages-artifact@v3
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

Três permissões que não podem faltar: `pages: write` (publicar),
`id-token: write` (provar a identidade do Pages) e `contents: read`.
O `--strict` transforma aviso de build em erro — o CI falha cedo em vez de
publicar site quebrado.

## Passo 6 — Ativar o Pages

Pela interface é em *Settings → Pages → Source → GitHub Actions*. Pela CLI:

```bash
gh api -X POST repos/jailsonh3/jailsonh3.github.io/pages -f build_type=workflow
```

Se já estiver ativado, atualize:

```bash
gh api -X PUT repos/jailsonh3/jailsonh3.github.io/pages -f build_type=workflow
gh api repos/jailsonh3/jailsonh3.github.io/pages --jq '"build_type: \(.build_type)"'
```

## Passo 7 — Publicar e verificar

```bash
git add -A && git commit -m "feat: site inicial" && git push
gh run watch          # acompanha o deploy
```

Conferindo se o site é o que você construiu (e não um build antigo):

```bash
curl -s https://jailsonh3.github.io/ | grep -oE 'generator" content="[^"]*"'
```

Esse `generator` é a forma mais rápida de descobrir **qual build está no ar**.
`zensical-0.1.1` = site bom. `Jekyll v3.10.0` = armadilha 1 acontecendo.

## Passo 8 — Publicar posts

Crie `docs/blog/posts/$(date +%F)-slug.md`:

```markdown
---
date: 2026-11-11
categories:
  - DevOps
tags:
  - github-pages
  - zensical
---

# Título do post

Resumo de uma ou duas frases — é o que aparece na listagem.

<!-- more -->

Conteúdo completo do post.
```

- `categories` → páginas de categoria (`/blog/category/linux/`)
- `tags` → filtros e índice em `/tags/`
- `<!-- more -->` → separa o resumo do post

Preview local antes de publicar:

```bash
uv tool run --from zensical zensical serve
```

## Passo 9 — Temas: o que dá para mudar

**Variantes.** `theme.variant` tem duas opções: `modern` (padrão, novo e mais
leve) e `classic` (aparência idêntica ao Material for MkDocs).

**Paletas.** As cores disponíveis para `primary`/`accent`:

```
red  pink  purple  deep purple  indigo  blue  light blue  cyan  teal
green  light green  lime  yellow  amber  orange  deep orange  brown
grey  blue grey  black  white
```

`scheme: default` é o modo claro e `slate` o escuro — dá pra definir os dois
em entradas separadas da lista `palette`, cada uma com seu `toggle`.

**Fontes.** Qualquer fonte do Google Fonts via `theme.font.text` e
`theme.font.code`. Para não carregar nada do Google (privacidade e velocidade):

```yaml
theme:
  font: false   # usa as fontes do sistema
```

**Ícones.** O tema novo usa ícones no formato `lucide/...` (ex.: `lucide/sun`,
`lucide/moon`). Dá para adicionar ícones próprios em `theme.icon`.

**CSS extra.** Para ajustes finos, um arquivo próprio:

```yaml
extra_css:
  - stylesheets/extra.css
```

```css
:root {
  --md-text-font: "Inter";
  --md-code-font: "JetBrains Mono";
}
```

## Armadilhas (a parte que custou mais tempo)

### 1. O build Jekyll fantasma sobrescreve o seu site

Se o repo se chama `<usuario>.github.io`, o GitHub o trata como **site de
usuário** e agenda um workflow próprio, `pages build and deployment`, usando
o build **legacy** (Jekyll). Ele roda em paralelo com o seu e, se terminar
depois, **publica o repositório cru por cima** do site gerado. O sintoma é
inquieto: o site funciona, some, volta, e os posts viram 404.

O conserto:

```bash
gh api -X PUT repos/<user>/<user>.github.io/pages -f build_type=workflow
```

E um arquivo `.nojekyll` na raiz, para o build legado não tentar processar
nada como Jekyll:

```bash
touch .nojekyll
```

Depois de corrigir, **um push de teste** para confirmar: na aba *Actions* só
deve aparecer o seu workflow.

### 2. Pages em repo privado exige plano pago

No plano Free, o GitHub Pages só funciona em repositório **público**. Se você
tentar ativar em repo privado, a API responde:

```
422 Your current plan does not support GitHub Pages for this repository.
```

Alternativas se a fonte precisa ser privada:

- **Híbrido**: repo privado com o código-fonte + repo público que recebe
  só o `site/` buildado (o push entre repos exige um token guardado como
  secret).
- **Cloudflare Pages / Netlify**: aceitam repo privado via GitHub App e têm
  plano grátis (URL em `pages.dev`).
- **GitHub Pro**: habilita Pages privado.

### 3. `source.path` engana

Depois de ativar, o campo `source` pode mostrar `/docs` — isso é o vestígio do
build legacy e **não** importa enquanto o `build_type` for `workflow`.

### 4. Cache e propagação

O Pages pode levar alguns minutos (e o navegador cacheia). Antes de entrar em
pânico:

```bash
curl -sI https://<user>.github.io/ | head -1
gh api repos/<user>/<user>.github.io/pages --jq '.status'
```

## Checklist final

- [ ] Repo criado e `main` como branch principal
- [ ] `mkdocs.yml` com `site_url` correto (senão o sitemap e os links ficam errados)
- [ ] Workflow com `pages: write` + `id-token: write`
- [ ] `build_type: workflow` (e `.nojekyll` se o repo é `<user>.github.io`)
- [ ] `zensical build --strict` passa local
- [ ] `curl` na URL pública responde `200` com o `generator` esperado
- [ ] Preview local conferido antes do push

## Comandos úteis de dia a dia

```bash
gh run list --limit 5                       # últimos deploys
gh run watch                                # acompanha o deploy atual
gh run view --log-failed                    # log da execução que falhou
gh api repos/<user>/<repo>/pages            # status e config do Pages
gh api repos/<user>/<repo>/pages --jq '.status'
```

## Referências

- [GitHub Pages — documentação](https://docs.github.com/pt/pages)
- [Zensical](https://zensical.org/) — sucessor mantido do MkDocs
- [MkDocs Material — plugins blog e tags](https://squidfunk.github.io/mkdocs-material/plugins/blog/)
- [GitHub CLI — `gh api`](https://cli.github.com/manual/gh_api)
