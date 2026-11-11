# Diário de um Dev

Blog técnico sobre bugs, soluções e curiosidades do dia a dia de quem programa.

🔗 **Site:** https://jailsonh3.github.io/

## Estrutura

```
.
├─ mkdocs.yml                    # configuração do site (compatível com MkDocs)
├─ .github/workflows/deploy.yml  # build + deploy no GitHub Pages
├─ docs/
│  ├─ index.md                   # página inicial
│  ├─ sobre.md                   # sobre o autor
│  ├─ tags.md                    # índice de tags
│  └─ blog/
│     ├─ index.md                # listagem de posts
│     └─ posts/                  # um arquivo Markdown por post
└─ site/                         # build local (ignorado pelo git)
```

## Publicar um post novo

1. Crie o arquivo em `docs/blog/posts/` com o nome no formato
   `AAAA-MM-DD-slug.md` (a data define a ordenação):

   ```bash
   cd docs/blog/posts
   touch $(date +%F)-meu-problema.md
   ```

2. Escreva o front matter e o conteúdo:

   ```markdown
   ---
   date: 2026-11-10
   categories:
     - Linux
   tags:
     - systemd
   ---

   # Título do post

   Resumo em uma ou duas frases.

   <!-- more -->

   Conteúdo completo.
   ```

   O `<!-- more -->` separa o resumo (usado nas listagens) do post completo.
   `categories` viram páginas de categoria; `tags` viram filtros na busca.

3. Teste localmente (com preview em <http://localhost:8000>):

   ```bash
   uv tool run --from zensical zensical serve
   ```

4. Publique:

   ```bash
   git add -A && git commit -m "post: meu problema" && git push
   ```

   O workflow `.github/workflows/deploy.yml` constrói o site e publica no
   GitHub Pages automaticamente (leva ~1 minuto).

## Build local sem servidor

```bash
uv tool run --from zensical zensical build --strict
# resultado em site/
```

O `--strict` transforma warnings em erro — é o mesmo comando usado no CI, então
o que passa local passa no GitHub Actions.

## Stack

- [Zensical](https://zensical.org/) — gerador estático (sucessor mantido do
  MkDocs, lê o mesmo `mkdocs.yml`)
- [GitHub Pages](https://pages.github.com/) — publicação via GitHub Actions
- Python + [uv](https://docs.astral.sh/uv/) — ambiente de build, sem instalar
  nada globalmente
