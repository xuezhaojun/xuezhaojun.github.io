# AGENTS.md

## Project

Hugo static site (personal blog). Theme: PaperMod (git submodule — never edit `themes/PaperMod/`).

## Commands

```bash
hugo server          # local dev server
hugo --gc --minify   # production build (used by CI)
```

No Makefile, no package.json, no other build tooling.

## Content

- **Posts** go in `content/posts/`. Every article lives here and shows up under Blog.
- **Collections** are a Hugo taxonomy (`collections`), not a content section. A post joins a collection via frontmatter, e.g. `collections: ["Go Concurrency"]` plus `weight: 1` for chapter ordering. `/collections/` lists all collections; each collection page shows its posts as a book-style TOC, and post pages get a collection badge + prev/next chapter navigation. Use title-case collection names (the URL is auto-slugged).
- **Tags** are a closed set — a post's `tags` must only contain values from this list, nothing else:
  - Languages: `golang`, `python`, `typescript`
  - Domains: `k8s`, `ai-infra`, `ai-agent`
  - Meta: `translated` (the post is translated by the author)
- All content uses **YAML frontmatter** (`---`), not TOML.
- Required frontmatter: `title`, `date`, `draft`, `tags`, `summary`.
- Site is bilingual (Chinese primary, English secondary).

## Gotchas

- Hugo version: **0.152.1 extended** (CI pins this exact version).
- `markup.goldmark.renderer.unsafe = true` — raw HTML in Markdown is intentional.
- Mermaid diagrams work via a custom code block renderer (`layouts/_default/_markup/render-codeblock-mermaid.html`).
- Images go in `static/images/`.
- Pushing to `main` triggers GitHub Pages deployment automatically.
