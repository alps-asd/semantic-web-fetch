# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** an application. It is a [Claude Skill](https://docs.claude.com) — `semantic-web-fetch` — together with a small demo site that proves what the skill does. There is no build, lint, or test tooling; the artifacts here are documents, a skill definition, and a static HTML page.

## The skill's core idea

The skill makes Claude read a web page **in conformance with the profile the page declares about itself** (ALPS / microformats / RDFa / microdata / schema.org), rather than inferring meaning from the rendered surface. It is defined by three principles, not a procedure — when editing `SKILL.md`, preserve this principle-first framing:

1. **Authority** — meaning is grounded in the declared profile, not in contextual inference.
2. **Binding** — only elements bindable to declared descriptors count as grounds for meaning; follow each descriptor's `def` (e.g. to schema.org) and include it as authority.
3. **No inferred bindings** — elements absent from the profile are reported as *unbound*, never filled in by guessing.

The behavioral consequences (fetch raw HTML not extracted text; also fetch the `rel="profile"` referent; recover affordances via ALPS `type` safe/idempotent/unsafe and `rt` target state) follow *from* the principles and should be kept downstream of them in the prose, not promoted to rules.

## File roles

- `semantic-web-fetch/SKILL.md` — the unpacked skill source (YAML frontmatter `name`/`description` + the principle-driven body). This is the source of truth for the skill's behavior. It lives in a folder named after the skill because that is the canonical skill layout (`<skill-name>/SKILL.md`) and because the packager uses the folder name as the archive's top-level directory.
- `semantic-web-fetch.skill` — the **packaged** skill: a ZIP whose single entry is `semantic-web-fetch/SKILL.md`. It must stay in sync with the source. Repackage with the official skill-creator packager (it validates the frontmatter, then zips the folder excluding `.DS_Store`, `__pycache__`, `*.pyc`, and a root-level `evals/`):
  ```bash
  SC=~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator
  # the packager needs pyyaml; this repo's system python lacks it, so use a throwaway venv
  python3 -m venv /tmp/skillcreator-venv && /tmp/skillcreator-venv/bin/pip install -q pyyaml
  (cd "$SC" && /tmp/skillcreator-venv/bin/python -m scripts.package_skill \
     /Users/akihito/git/semantic-web-fetch/semantic-web-fetch \
     /Users/akihito/git/semantic-web-fetch)
  ```
- `index.html` + `registry.alps.json` — the demo. The HTML's surface (column headers "Date"/"Origin", the intro prose, the "Featured" chip) is **deliberately misleading**; `registry.alps.json` is the ALPS profile that overrides it. The page links the profile via `<link rel="profile" href="registry.alps.json">`. These two files are a matched pair: any descriptor renamed in the ALPS must match the corresponding `class` in the HTML, and vice versa.
- `README.md` — explains the skill and walks through the demo's "answer key" (surface vs. profile-defined meaning).

## Demo invariants (do not break)

The demo only works as a teaching tool if the surface and the profile *disagree* in specific ways. When editing either file, keep these intact:

- `entryDate` (class on the "Date" column) → `schema.org/datePublished` = listing date, **not** manufacture date.
- `displayCity` ("Origin" column) → `schema.org/contentLocation` = current showroom, **not** place of origin.
- The "Featured" chip has **no** descriptor — it must remain unbound.
- Per row there are two affordances: `goRecord` (`type: safe`, ViewAction) on the object name, and `reserveObject` (`type: unsafe`, ReserveAction) on the Reserve control. The HTML markup must not itself signal which is safe — only the profile does.

## Serving the demo

The demo is published via **GitHub Pages** (branch `main`, folder `/ (root)`); that hosted URL is the canonical way to try it, and it is what you point Claude at. Pages serving keeps the relative `registry.alps.json` profile link resolving — a `file://` open would break it.

## Publishing note

Everything lives at the **repository root** — `index.html`, `registry.alps.json`, and `SKILL.md` are not nested under `docs/` or `skill/`. GitHub Pages should therefore be served from branch `main`, folder `/ (root)`, and `README.md` references the root paths accordingly. Keep the layout flat unless you also update the README and the `.skill` packaging step.

The demo's `goRecord` / `reserveObject` links target `record.html` / `reserve.html`, which are intentionally **not** present — they are placeholders. The skill demonstrates affordance *typing* and `rt` target state, not working navigation, so do not "fix" these as broken links.
