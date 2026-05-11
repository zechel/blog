# Project Brief — josezechel.com

| Field | Value |
|---|---|
| **From** | @architect (Aria) |
| **To** | @pm (Morgan) |
| **Purpose** | Input for the formal PRD — please consume this brief, elicit any gaps with the owner, and produce `docs/prd/blog-prd.md`. |
| **Project** | josezechel.com (personal blog) |
| **Owner** | José Zechel |
| **Date** | 2026-05-11 |
| **Status** | Ready for PM elaboration |
| **Companion doc** | `docs/architecture/blog-architecture.md` (already drafted) |

---

## 1. Elevator Pitch

A personal blog by José Zechel — minimalist, editorial, pt-BR — where he publishes essays on AI, automation, and infrastructure for builders in the Brazilian tech community. The site is engineered to be low-maintenance (static, free hosting) and feel writerly rather than corporate.

## 2. Problem & Opportunity

**Problem:** José writes long-form thinking that doesn't fit LinkedIn posts or Twitter threads. The current state is "no permanent home for ideas" — drafts live in private notes; conversations happen in DMs and get lost.

**Opportunity:**
- A canonical URL per essay that can be linked from talks, LinkedIn, podcasts, and email signatures.
- A discoverable archive that compounds in SEO value over time.
- A comment system anchored in GitHub identity to attract serious replies, not drive-by reactions.
- A LinkedIn CTA at the end of every post that converts readers into followers / DMs (already implemented).

## 3. Target Audience

**Primary:** Mid-to-senior engineers and tech leads in Brazil who build with AI, automation, or platform infrastructure. They read in pt-BR, are comfortable with English source material, and value depth over hot-takes.

**Secondary:** International readers reaching essays via search or shares. (Site is monolingual pt-BR for v1; English is a future option.)

**Anti-audience:** Beginners looking for tutorials; the editorial voice assumes context.

## 4. Goals

### Business / personal-brand goals
- Establish José as a recognizable voice on AI + infra in pt-BR.
- Generate inbound for speaking, consulting, and collaboration via the per-post LinkedIn CTA.
- Own the canonical URL for his ideas (no platform risk).

### User goals
- Find an essay quickly (search + archive + tags).
- Read comfortably on phone or desktop, light or dark.
- Comment without creating a new account (GitHub auth via Giscus).
- Subscribe to updates via RSS (link in nav).

### Technical goals (constraints carried over from architecture)
- Zero ongoing infra cost.
- Sub-second TTFB.
- Lighthouse ≥ 95.
- Author publishes by `git push` — no CMS to maintain.

## 5. Success Metrics

Suggested for PM to refine and prioritize:

| Metric | Target (first 6 months) |
|---|---|
| Posts published | ≥ 12 (one every ~2 weeks) |
| Unique readers / month | TBD — needs analytics decision (currently none) |
| Comments per post (median) | ≥ 1 |
| LinkedIn follower delta attributable to CTA | TBD — needs UTM strategy |
| Lighthouse Performance | ≥ 95 sustained |

**Open question for @pm:** does José want analytics at all? Privacy posture is currently "no third-party trackers." If yes → recommend Plausible self-hosted; if no → success measured by Discussions activity + LinkedIn referrals only.

## 6. Scope

### In scope for v1 (already built or in flight)
- [x] Hugo + Hextra static site, pt-BR
- [x] Custom design system (Lora serif + Geist Mono, accent `#0082fb`, light/dark)
- [x] Posts section with tags
- [x] About page (`/sobre`)
- [x] Archive page (`/arquivo`)
- [x] RSS feed (in nav)
- [x] Client-side search (FlexSearch)
- [x] Giscus comments (config in place; needs manual GitHub steps)
- [x] LinkedIn CTA at the end of every post
- [x] GitHub Actions deploy pipeline
- [x] Footer with theme toggle + language label + Hextra credit
- [ ] Custom domain `josezechel.com` (DNS pending)
- [ ] Aliases `josezechel.com.br`, `zechel.com.br` redirecting to apex (registrar config pending)

### Out of scope for v1 (candidates for v2+)
- Newsletter / email subscription
- Series or multi-part collections
- Multi-language content
- Self-hosted analytics
- Reading-time estimates per post
- Related-posts widget
- Author bio block above each post

## 7. Key Features (proposed; PM to confirm/expand into PRD)

1. **Essay reading experience** — single-column editorial layout, 680px max-width, serif body.
2. **Per-post comments via Giscus** — lazy-loaded, GitHub auth.
3. **LinkedIn CTA card** — appears at the bottom of every post, before comments. Static copy currently: "Gostou deste texto?" + button to José's LinkedIn.
4. **Archive view** — chronological list of all posts by year.
5. **Tag pages** — every tag in front-matter generates a tag index automatically.
6. **Search** — overlay search input in nav, in-browser FlexSearch.
7. **RSS** — `/index.xml`, link visible in nav.
8. **Theme toggle** — light / dark / system, persists in localStorage.
9. **About page** — short bio at `/sobre`.

## 8. Constraints

| Constraint | Type | Note |
|---|---|---|
| pt-BR only for v1 | Product | English is a deferred option, not a near-term requirement. |
| No third-party trackers | Privacy | José's stance. Re-check during PRD pass. |
| Free hosting (GitHub Pages) | Cost | Holds until traffic exceeds Pages' soft 100 GB/month limit — not a concern at projected volume. |
| Hextra theme constraints | Technical | We can override layouts but should not fork Hextra. Major Hextra version bumps will require manual diff. |
| Author publishes by Git | Workflow | No CMS. Posts are Markdown files. Onboarding non-technical guest authors would require rethinking. |
| Comments require GitHub account | UX | Acceptable for engineer audience; would be a barrier for general audiences (not the target). |

## 9. Assumptions

- José is comfortable with a Markdown-in-Git authoring flow.
- Audience reach is built primarily via LinkedIn, not SEO (CTA strategy reflects this).
- No commercial transactions ever happen on-site (so no need for payments, accounts, etc.).
- Comment moderation overhead is low enough that José handles it directly via GitHub Discussions UI.

## 10. Open Questions for @pm

1. **Analytics**: any analytics at all? If yes, which tool and what privacy guarantees do we commit to?
2. **Newsletter**: ship in v1 or v2? If v1, which provider (Buttondown vs. Resend self-rolled vs. Substack-as-archive)?
3. **Comment policy**: do we publish a code-of-conduct page? What triggers moderation?
4. **Series / multi-part essays**: is this a near-term need?
5. **Author bio block**: should it appear above each post, in the footer, or only on `/sobre`?
6. **Related posts**: nice-to-have or v1 requirement?
7. **Reading time**: include estimate at top of posts?
8. **CTA copy**: the current LinkedIn card uses a static "Gostou deste texto?" — should the copy vary by post category, or stay static?
9. **English version**: any near-term plans, or genuinely deferred?
10. **SEO target keywords**: should we define a content strategy / keyword cluster, or write purely from author voice?

## 11. Handoff Notes & Next Steps

**For @pm (Morgan):**
1. Read the companion Architecture Doc (`docs/architecture/blog-architecture.md`) so the PRD doesn't contradict the build.
2. Run `*gather-requirements` with José using this brief as the starting input — focus on the Open Questions in §10.
3. Produce `docs/prd/blog-prd.md` using `prd-tmpl.yaml`.
4. After PRD is approved, hand off to @po for story breakdown.

**For the owner (José):**
The remaining manual steps to bring the site live are listed in §13.1 of the Architecture Doc (GitHub repo + Pages + Discussions + Giscus + DNS). None of these block the PRD pass.

**For @architect (Aria):**
Standing by for any architecture follow-ups raised during PRD elaboration — e.g., if analytics is approved, we'll need an ADR for the chosen tool.

---

*End of Project Brief — over to you, Morgan. — Aria 🏗️*
