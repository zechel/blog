# josezechel.com — Architecture Document

| Field | Value |
|---|---|
| **Project** | josezechel.com (personal blog) |
| **Owner** | José Zechel |
| **Author** | @architect (Aria) |
| **Status** | Draft v1 |
| **Last updated** | 2026-05-11 |
| **Source of truth** | This document |

> Scope: covers system design, tech stack, build/deploy pipeline, and operations. Product strategy / requirements live in the Project Brief (`docs/briefs/project-brief.md`) and, once produced, in the PRD (`docs/prd/`).

---

## 1. Overview

A static personal blog at `https://josezechel.com/`, built with Hugo + the Hextra theme, hosted on GitHub Pages, with a minimalist editorial design (serif body, monospaced UI, single accent color). Comments are federated through GitHub Discussions via Giscus. The author publishes essays in Brazilian Portuguese (pt-BR) about AI, automation, and infrastructure.

The system is intentionally low-operations: no runtime backend, no database, no CMS — content is Markdown in Git, builds are deterministic, hosting is free.

---

## 2. Tech Stack

| Layer | Choice | Version | Rationale |
|---|---|---|---|
| **Static site generator** | Hugo Extended | 0.161.1 (min 0.146.0) | Fast builds (<200 ms for the whole site), Go-templated layouts, mature ecosystem. Extended variant required for SCSS and image processing used by Hextra. |
| **Theme** | Hextra | v0.12.3 | Modern Tailwind-based Hugo theme with built-in dark mode, FlexSearch, i18n, comments support. Pulled via Hugo Modules so updates are a single `go.mod` bump. |
| **Module resolver** | Go | 1.26.0 | Required by `hugo mod tidy` to resolve Hextra. Installed locally at `~/.local/go`, not bundled with the site. |
| **Hosting** | GitHub Pages | n/a | Free, integrated with the repo, custom-domain support, automatic HTTPS. No vendor lock-in (static artifact is portable to Netlify/Cloudflare Pages). |
| **CI/CD** | GitHub Actions | n/a | Builds and deploys on push to `main`. Defined in `.github/workflows/pages.yaml`. |
| **Comments** | Giscus | latest | Comments stored as GitHub Discussions in the repo. Zero infra, free, abuse-resistant (auth via GitHub). |
| **Search** | FlexSearch (in-browser) | bundled with Hextra | Client-side full-text index over all pages; no backend. |
| **Fonts** | Google Fonts: Lora (serif), Geist + Geist Mono | n/a | Lora for editorial body; Geist Mono for nav/meta/code — establishes the minimalist serif-vs-mono contrast. Loaded with `preconnect` to minimize FOUT. |
| **Color tokens** | CSS custom properties | n/a | Light + dark themes via `:root` + `.dark`. Accent: `#0082fb` (Meta blue). Zero shadows, zero gradients. |
| **Markdown engine** | Goldmark (Hugo built-in) | n/a | `unsafe: true` enabled so authors can embed HTML in posts. |
| **Syntax highlighting** | Chroma (Hugo built-in) | n/a | Style `github-dark` for code fences. |
| **Domain** | `josezechel.com` | — | Primary apex domain. Aliases (`josezechel.com.br`, `zechel.com.br`) 301-redirect to apex. |

---

## 3. System Topology

```
              ┌─────────────────────────────────────────┐
              │            Author (José)                │
              └─────────────────┬───────────────────────┘
                                │ git push
                                ▼
              ┌─────────────────────────────────────────┐
              │   GitHub repo: zechel/josezechel-blog   │
              │   ─ content/posts/*.md  (source)        │
              │   ─ hugo.yaml           (config)        │
              │   ─ layouts/*           (overrides)     │
              └─────────────────┬───────────────────────┘
                                │ on push to main
                                ▼
              ┌─────────────────────────────────────────┐
              │   GitHub Actions workflow (pages.yaml)  │
              │   1. setup-hugo @ HUGO_VERSION          │
              │   2. hugo --gc --minify                 │
              │   3. upload-pages-artifact (public/)    │
              │   4. deploy-pages                       │
              └─────────────────┬───────────────────────┘
                                │
                                ▼
              ┌─────────────────────────────────────────┐
              │   GitHub Pages CDN                      │
              │   └─ https://josezechel.com/            │
              └─────────────────┬───────────────────────┘
                                │ HTTPS (Let's Encrypt, auto-renewed)
                                ▼
              ┌─────────────────────────────────────────┐
              │            Reader's browser             │
              │   ─ Renders static HTML/CSS/JS          │
              │   ─ Lazy-loads Giscus iframe on /posts/ │
              │   ─ Runs FlexSearch client-side         │
              └──────────┬──────────────────────────────┘
                         │
                         │ (comment write)
                         ▼
              ┌─────────────────────────────────────────┐
              │   GitHub Discussions (Comments cat.)    │
              │   ─ One discussion per post (mapping:   │
              │     pathname)                           │
              └─────────────────────────────────────────┘
```

There is **no application server** in the loop at any point. The reader's browser talks only to: (a) GitHub Pages CDN for HTML/CSS/JS, (b) Google Fonts for typefaces, (c) GitHub's Giscus iframe for comments.

---

## 4. Build & Deploy Pipeline

**Trigger:** `push` to `main` or manual `workflow_dispatch`.

**Workflow** (`.github/workflows/pages.yaml`):

1. `actions/checkout@v4` with `submodules: recursive` and `fetch-depth: 0` (last is required for `enableGitInfo: true` to compute Lastmod from commit history).
2. `peaceiris/actions-hugo@v3` pinned to `HUGO_VERSION: 0.161.1` with `extended: true`.
3. `hugo --gc --minify --baseURL "https://josezechel.com/"` — produces `public/`.
4. `actions/upload-pages-artifact@v3` uploads `public/`.
5. `actions/deploy-pages@v4` flips the live site.

**Concurrency:** `group: pages` with `cancel-in-progress: true` — overlapping deploys for the same branch cancel earlier runs.

**Permissions (least-privilege):**
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

**Local-vs-CI parity:** `HUGO_VERSION` is the single source of truth for both environments. When bumping locally, also bump the env var in the workflow.

**Build artifacts (not committed):** `public/`, `resources/`, `.hugo_build.lock` — all in `.gitignore`.

---

## 5. Content Model

```
content/
├── _index.md             # Home (lists recent posts)
├── posts/
│   ├── _index.md         # Section index; cascade.type: blog → triggers
│   │                       layouts/blog/single.html for every post
│   └── ola-mundo.md      # First post (and template for future posts)
├── sobre.md              # About page
└── arquivo/
    └── _index.md         # Archive index (all posts grouped by year)
```

**Front-matter contract** for posts:

```yaml
---
title: "Post title"
date: 2026-05-10           # Required — used by Hextra for sort + meta
lastmod: 2026-05-11        # Optional — overrides Git lastmod
tags: [meta, manifesto]    # Drives Hextra tag pages
draft: false               # Set true to exclude from build
comments: true             # Per-page Giscus toggle
description: "SEO summary" # OG description + meta tag
---
```

**Why `cascade.type: blog`** on `posts/_index.md`: Hugo resolves single-page layouts in the order `layouts/<type>/single.html` → `layouts/<section>/single.html` → `layouts/_default/single.html`. Setting `type: blog` via cascade tells Hugo to look up `layouts/blog/single.html` first — this is the override file where I injected the LinkedIn CTA partial directly before the comments partial. Without the cascade, Hextra's own `blog/single.html` would win and the CTA wouldn't render.

---

## 6. Theming & Design System

### 6.1 Typography
- **Body:** Lora (variable serif, 400/500/600/700 + italic).
- **UI / nav / meta / code:** Geist Mono.
- **Headings:** Geist Mono (small caps feel) — overrides Hextra's default sans.
- Loaded via `<link rel="preconnect">` + Google Fonts CSS, injected in `layouts/partials/custom/head-end.html`.

### 6.2 Color tokens
Defined as CSS custom properties in `assets/css/custom.css`:

| Token | Light | Dark |
|---|---|---|
| `--hextra-bg` | `#ffffff` | `#0d0f12` |
| `--hextra-text` | `#1a1a1a` | `#e5e5e5` |
| `--hextra-text-muted` | `#5a5a5a` | `#9ca3af` |
| `--hextra-border` | `#e5e5e5` | `#2a2d34` |
| `--hextra-accent` | `#0082fb` | `#0082fb` (unchanged) |

**No shadows, no gradients** — all separation comes from `1px` borders in `--hextra-border`. This is an explicit aesthetic constraint, not a default.

### 6.3 Layout
- Content column: `max-width: 680px`, centered.
- Vertical rhythm: `1.7rem` line-height on body, `2.5rem` between paragraphs.
- Heading scale: `h1 2.2rem` → `h2 1.6rem` → `h3 1.2rem`.
- Fade-in animation on `main`: `opacity 0 → 1` over 300ms on page-load — the only motion in the entire site.

### 6.4 Custom layouts
Files under `layouts/` that override Hextra defaults:

| File | Purpose |
|---|---|
| `layouts/blog/single.html` | Verbatim copy of Hextra's `blog/single.html` with `{{ partial "custom/postFooter.html" . }}` injected before comments. |
| `layouts/_partials/footer.html` | Footer override: forces switches row (theme + language + powered-by) to always be visible, with a static "Português" label since the site is monolingual. |
| `layouts/partials/custom/head-end.html` | Font preconnect + Google Fonts link + minified+fingerprinted custom CSS + `<meta name="theme-color">`. |
| `layouts/partials/custom/postFooter.html` | LinkedIn CTA card rendered at the bottom of every post. |

---

## 7. Comments Subsystem (Giscus)

**Flow:**
1. Reader scrolls to bottom of a post.
2. Hextra's `components/comments.html` partial lazy-loads the Giscus `<script>`.
3. Giscus iframe initializes; on first comment it auto-creates a Discussion in the `Comments` category, mapped by URL pathname.
4. Subsequent visits hydrate from the existing Discussion.

**Configuration** (in `hugo.yaml` → `params.comments.giscus`):
- `repo: zechel/josezechel-blog`
- `repoId: PREENCHER_DEPOIS` — **manual step**, retrieved from `giscus.app` configurator after enabling Discussions.
- `category: Comments` (created manually with type "Announcements" recommended).
- `categoryId: PREENCHER_DEPOIS` — same source as repoId.
- `mapping: pathname` — robust against title changes; breaks if URLs change (mitigate via Hugo `aliases:`).
- `lightTheme: light` / `darkTheme: dark_dimmed` — picked to match site palette.
- `loading: lazy` — defers Giscus JS until iframe enters viewport.

**Per-page toggle:** `comments: false` in a post's front matter hides the section.

**Spam/abuse mitigation:** Authentication is delegated to GitHub. Moderate by editing/deleting Discussions in the repo.

---

## 8. i18n & Localization

The site is currently **monolingual pt-BR**. The infrastructure is multilingual-ready (Hugo language map exists with one entry), but no en/es languages are configured.

**Translation override:** `i18n/pt-br.yaml` ships project-specific strings (Footer copy, "Powered by", "Light/Dark/System", "Change language" etc.). This overrides Hextra's bundled `pt.yaml` (which is pt-PT, not pt-BR).

**Language detection:** Hugo prefers the more-specific code (`pt-br`) over the generic (`pt`).

**To add a second language** (future): add a new entry under `languages:` in `hugo.yaml`, mirror `content/` under `content/en/`, create `i18n/en.yaml`. The footer override already has a guarded multilingual branch.

---

## 9. Search

**Tech:** Hextra's built-in FlexSearch integration.

**Build-time:** Hugo emits a JSON index from all pages with `content` tokenized via `forward` n-grams (`tokenize: forward`).

**Runtime:** The search input in the navbar opens an overlay; FlexSearch runs in-browser, no network. Index size scales linearly with content — at ~50 posts the JSON is ≈100 KB gzipped.

**Trade-off:** No relevance tuning beyond FlexSearch's defaults. For a personal blog this is fine; if the corpus grows past several hundred posts, consider migrating to Pagefind or Algolia DocSearch.

---

## 10. SEO & Analytics

**SEO:**
- `params.seo.enable: true` activates Hextra's OpenGraph + Twitter Card meta-tag generation.
- Per-page `description` front-matter populates `<meta name="description">` and `og:description`.
- `enableGitInfo: true` exposes `Lastmod` from Git commit history → consumed by Hugo's sitemap.xml.
- `sitemap.xml` and `index.xml` (RSS) are generated automatically by Hugo at the site root.

**Analytics:** None currently. Privacy stance is no third-party trackers. If analytics is added in the future, recommend self-hosted Plausible or Umami — never GA.

---

## 11. Performance Budget

| Metric | Budget | Strategy |
|---|---|---|
| First contentful paint | < 1.0 s on 4G | Static HTML, critical CSS inlined via Hugo `resources.Minify` + fingerprinting |
| Total page weight (post page) | < 200 KB | No images by default; Lora WOFF2 ≈ 60 KB; Geist Mono WOFF2 ≈ 35 KB; CSS ≈ 15 KB; Giscus deferred |
| Lighthouse Performance | ≥ 95 | Achieved by absence — no JS framework, no analytics, no above-fold images |
| Cumulative Layout Shift | < 0.05 | Fonts use `font-display: swap`; reserve space for Giscus iframe |

---

## 12. Security Considerations

**Threat model is light:** the site has no auth, no PII storage, no admin surface. Realistic risks:

| Risk | Mitigation |
|---|---|
| Comment spam / abuse | Giscus → GitHub auth required to post. Author moderates via Discussions UI. |
| DNS hijack / cert misissuance | DNSSEC at registrar; HSTS via Pages settings; CAA record allowing only Let's Encrypt. |
| Supply-chain (Hextra) | Hextra is pinned in `go.mod` to v0.12.3 + checksums in `go.sum`. Bumps reviewed manually. |
| Supply-chain (Google Fonts) | Subresource Integrity is impractical for Google Fonts CSS (URL not stable). Risk accepted; alternative: self-host the WOFF2 files. |
| Secret leakage | `.env` is gitignored; no secrets needed at build time. |
| Cross-site scripting in posts | `goldmark.renderer.unsafe: true` allows raw HTML in Markdown. Mitigated by single-author trust model; would need rethinking if guest posts are accepted. |

---

## 13. Operational Runbook

### 13.1 Initial bring-up (manual steps remaining)

1. **GitHub repo + first push** — `gh auth login` → `gh repo create zechel/josezechel-blog --public --source=. --remote=origin` → commit + push.
2. **Enable Pages** — Settings → Pages → Source: **GitHub Actions**.
3. **Enable Discussions** — Settings → Features → check "Discussions". Create a category named **Comments** (type: Announcements recommended).
4. **Install Giscus app** — visit `github.com/apps/giscus` → install on `zechel/josezechel-blog`.
5. **Configure Giscus** — visit `giscus.app`, fill the repo, copy `data-repo-id` and `data-category-id`, paste into `hugo.yaml` replacing both `PREENCHER_DEPOIS` placeholders.
6. **DNS** — at the registrar for `josezechel.com`:
   - `A @ 185.199.108.153`
   - `A @ 185.199.109.153`
   - `A @ 185.199.110.153`
   - `A @ 185.199.111.153`
   - `CNAME www zechel.github.io`
7. **Custom domain in Pages** — Settings → Pages → Custom domain: `josezechel.com` → **Enforce HTTPS** on (wait for cert provisioning, ~15 min).
8. **CNAME file** — `static/CNAME` containing `josezechel.com` (Hugo copies it verbatim to `public/`).
9. **Domain aliases** — at the registrars for `josezechel.com.br` and `zechel.com.br`, set up a 301 redirect to `https://josezechel.com/` (most registrars expose this in the DNS UI).

### 13.2 Day-2 operations

| Task | How |
|---|---|
| New post | `content/posts/<slug>.md` with front-matter → commit + push → deploy is automatic |
| Local preview | `hugo server -D` at `:1313`, live-reload on save |
| Bump Hextra | `go get github.com/imfing/hextra@latest && hugo mod tidy` |
| Bump Hugo | Update `HUGO_VERSION` in `pages.yaml` and local install |
| Moderate comments | Repo → Discussions tab |
| Rotate accent color | Edit `--hextra-accent` in `assets/css/custom.css` (single source of truth) |

---

## 14. Risks & Future Work

| Item | Risk level | Notes |
|---|---|---|
| Hextra major-version bump may break layout overrides | Medium | Mitigated by pinning. Run visual diff before merging bumps. |
| FlexSearch index size beyond ≈500 posts | Low (long-horizon) | Migrate to Pagefind. |
| GitHub Pages bandwidth soft limit (100 GB/month) | Very low | Static blog, no images at scale. |
| Comments scaling — single GitHub repo per post mapping | Low | Discussion limit per repo is very high; not a real concern. |
| Loss of GitHub account → loss of comments | Medium | Periodic export of Discussions via GraphQL API is a hedge. |

**Out of scope (v1):**
- Newsletter / email subscription.
- TOC sidebar on post pages (currently `params.toc` is enabled but only renders if posts include H2+).
- Multi-language content (infra is ready; no translations planned).
- Series / collections.
- Self-hosted analytics.

---

## 15. Decision Log (ADRs)

| # | Decision | Alternatives considered | Rationale |
|---|---|---|---|
| ADR-1 | Hugo over Astro / Eleventy | Astro, Eleventy, Next.js static | Hugo is fastest, has Hextra ready-made, zero JS framework in output, and a single Go binary deploys. |
| ADR-2 | Hextra over PaperMod / Congo | PaperMod, Congo, Doks | Hextra is the most actively maintained Tailwind-based theme with built-in dark mode, FlexSearch, and i18n. Modules-based install (no submodules). |
| ADR-3 | Giscus over Disqus / Utterances / self-hosted Commento | Disqus (ads), Utterances (issues-based, polluting), Commento (hosting cost) | Giscus uses Discussions (cleaner separation from issues), free, no ads, GitHub-authenticated. |
| ADR-4 | GitHub Pages over Netlify / Cloudflare Pages | Netlify, Cloudflare Pages, Vercel | Free, in the same place as the source, no separate account, sufficient performance for personal blog. |
| ADR-5 | FlexSearch over Pagefind / Algolia | Pagefind, Algolia DocSearch | Bundled with Hextra; zero extra setup; acceptable up to several hundred posts. |
| ADR-6 | Lora + Geist Mono over Inter + system serif | various | Editorial serif body + monospaced UI establishes a distinct minimalist voice; Geist Mono in particular signals "engineer's blog" visually. |
| ADR-7 | No shadows, no gradients | Soft shadows on cards | Editorial aesthetic — separation by border is sharper and ages better. |
| ADR-8 | Comments lazy-loaded | Eager iframe | Performance: defers ~100 KB of Giscus JS until in-viewport. |
| ADR-9 | `mapping: pathname` for Giscus | `title`, `og:title` | Pathnames are stable as long as `aliases` is used on rename; titles aren't. |
| ADR-10 | LinkedIn CTA injected via cloned `blog/single.html` | Hextra hook, JS injection | Cascade `type: blog` makes Hextra's own layout win; cloning is the cleanest override. Documented so a future Hextra bump prompts a manual re-merge. |

---

*End of Architecture Document — Aria, arquitetando o futuro 🏗️*
