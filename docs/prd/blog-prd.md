# josezechel.com Product Requirements Document (PRD)

| Field | Value |
|---|---|
| **Project** | josezechel.com (personal blog) |
| **Owner** | José Zechel |
| **Author** | @pm (Morgan) |
| **Status** | Draft v0.1 — Section 1 of 8 |
| **Source inputs** | `docs/briefs/project-brief.md`, `docs/architecture/blog-architecture.md` |
| **Last updated** | 2026-05-11 |

---

## 1. Goals and Background Context

### 1.1 Goals

- Establish José Zechel as a recognizable voice on AI, automation, and infrastructure in pt-BR.
- Own a canonical URL per essay so ideas live on a permanent home (no platform risk).
- Convert engaged readers into LinkedIn followers and inbound conversations via a per-post CTA.
- Foster substantive comment threads anchored in GitHub identity (Giscus + Discussions).
- Ship and operate the site at zero ongoing infrastructure cost.
- Sustain Lighthouse Performance ≥ 95 and sub-second first contentful paint.
- Publish on a regular cadence (target: one essay every ~2 weeks for the first 6 months).
- Keep authoring workflow at "git push and forget" — no CMS to maintain.

### 1.2 Background Context

José currently writes long-form thinking that doesn't fit LinkedIn posts or Twitter threads — drafts live in private notes and substantive conversations dissolve into DMs. The Brazilian tech community has a thin layer of senior-engineer essay writing on AI/infra topics in pt-BR specifically, which leaves an opening for a credible voice in that niche. The site is engineered to be writerly rather than corporate: editorial typography, single accent color, no shadows, no gradients, light/dark via tokens.

The technical posture deliberately trades flexibility for permanence and frictionlessness. A static Hugo build hosted on GitHub Pages means: no servers to maintain, no monthly bills, no vendor lock-in, deterministic builds, and a clean separation between content (Markdown in Git) and presentation (Hextra theme + custom overrides). Comments use Giscus over GitHub Discussions — sufficient for an engineer audience, zero infra, and naturally moderated by the auth barrier. The companion Architecture Doc enumerates the full stack decision-by-decision and is the source of truth for "how it's built"; this PRD owns "what we're building and why".

### 1.3 Change Log

| Date | Version | Description | Author |
|---|---|---|---|
| 2026-05-11 | 0.1 | Initial draft — Goals & Background Context populated from brief | @pm (Morgan) |
