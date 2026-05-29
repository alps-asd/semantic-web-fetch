# semantic-web-fetch

A [Claude Skill](https://docs.claude.com) for reading a web page **in conformance with the profile
it declares about itself** — deferring the authority over meaning to that declaration instead of
inferring it from the rendered surface.

By default, when Claude is given a URL it reads extracted text and infers what things mean. That is
usually good enough, but the inference is guesswork, and the rendered surface can be wrong, ambiguous,
or actively misleading. When a page declares a semantic profile (ALPS, microformats, RDFa, microdata,
schema.org), this skill treats that declaration as the source of truth.

## The three principles

The skill is defined by principles, not a procedure. The procedure follows from them.

1. **Authority** — meaning is grounded in the declared profile, not in Claude's contextual inference.
2. **Binding** — only elements that can be bound to declared descriptors count as grounds for meaning;
   `def` references (e.g. to schema.org) are followed and included as authority.
3. **No inferred bindings** — elements absent from the profile are reported as *unbound*, never filled
   in by inference. What was determined from the definition is never conflated with what was guessed.

It also recovers **affordances** the extracted surface loses: which links are operations, of what kind
(safe / idempotent / unsafe), and which state each transitions to.

## `semantic-web-fetch` vs. plain `web_fetch`

Claude's default `web_fetch` and this skill read the *same URL* but answer different questions.
`web_fetch` answers *"what does this page appear to say?"*; `semantic-web-fetch` answers *"what does
this page declare that it means?"*

| | plain `web_fetch` (default) | `semantic-web-fetch` |
| --- | --- | --- |
| What is read | extracted text (Markdown-equivalent) | raw HTML **plus** the declared profile |
| Source of meaning | inference from the surface | the profile's descriptors (the authority) |
| Headings & prose | taken at face value | treated as surface; overridden where the profile disagrees |
| Ambiguous values | guessed from context | resolved by the descriptor's `def` |
| Decorative / unlabelled bits | may be absorbed as data | reported as **unbound**, never invented |
| Links | undifferentiated URLs | typed operations (safe / idempotent / unsafe) with a target state (`rt`) |
| Failure mode | confident but possibly wrong | declines to assert beyond the definition |
| Stability | breaks when the layout changes | stable as long as the profile holds |

### The same row, read both ways

Row `HV·001` of the demo. What the surface yields as plain text:

> **Brass Astrolabe** — made **1710**, originated in **Florence**, maker **G. Vanni**, *Featured*.
> Links: the name, and *Reserve*.

Conformed to the declared profile:

> **Brass Astrolabe** (`itemTitle`). `entryDate` = 1710 → the date it was **listed in the catalogue**
> (`datePublished`), *not* when it was made; the period of manufacture is **absent from the page**.
> `displayCity` = Florence → the **current showroom** (`contentLocation`), *not* origin; origin is
> **absent**. `makerNote` = G. Vanni. "Featured" → **unbound** (decorative). Affordances: the name is
> `goRecord` (**safe**, view → record page); the *Reserve* control is `reserveObject` (**unsafe**,
> `ReserveAction` → reservation hold).

Same bytes over the wire; opposite conclusions about what is true, what is missing, and what is safe
to click.

## Install

This repository ships the skill two ways:

- `semantic-web-fetch.skill` — the packaged skill, ready to upload/install.
- `semantic-web-fetch/SKILL.md` — the unpacked source.

Place it where your Claude environment loads skills, or upload the `.skill` package.

## Try it (the demo is deliberately misleading)

This repository's root doubles as a tiny site whose **surface lies**, so that the difference between
*inferring* and *conforming* becomes visible. It is published with GitHub Pages.

- `index.html` — an ordinary-looking instrument catalogue.
- `registry.alps.json` — the ALPS profile it declares (the authority).

The catalogue's `goRecord` / `reserveObject` links point at `record.html` / `reserve.html`, which are
**not** shipped here — they are placeholders. The demo's point is the *typing* of each affordance
(safe vs. unsafe) and its declared target state (`rt`), not working navigation; `rt` names the state a
transition leads to, while `href` is merely where the link would go.

**Published (GitHub Pages):** enable Pages under *Settings → Pages → Build and deployment →
Deploy from a branch*, branch `main`, folder `/ (root)`. The demo is then served at:

```
https://<your-username>.github.io/<your-repo>/
```

That published URL is what you point Claude at — no local server is needed, and serving over
GitHub Pages keeps the relative profile link (`registry.alps.json`) resolving as it does in production.

Then ask Claude, against that URL:

> Read this page in conformance with its declared profile. Do not fill gaps by inference.

A naive read (and the page's own prose) will get several things wrong. Conforming to the profile
corrects them. Here is the answer key.

### What the surface says vs. what the profile defines

| On the surface | Naive inference | What the profile actually defines |
| --- | --- | --- |
| Column header **"Date"**, prose says "the period in which each piece was made" | "Brass Astrolabe was *made* in 1710" | `entryDate` → `schema.org/datePublished`: the date it was **listed in the catalogue**. The period of manufacture is **not on the page at all.** |
| Column header **"Origin"**, prose says "city where it was crafted" | "Made in Florence" | `displayCity` → `schema.org/contentLocation`: the city where it is **currently displayed**. The place of origin is **not on the page.** |
| **"Featured"** tag in some rows | "Featured" is a data field | No descriptor exists for it → **unbound**, decorative. Do not treat it as data. |
| Two affordances per row — the **object name** (a link) and a small **"Reserve"** control | "A title link and a reserve button; probably both harmless navigation" | `goRecord` → `type: safe` (`ViewAction`, read-only) vs `reserveObject` → `type: unsafe` (`ReserveAction`, **state-changing**). The markup types neither; only the profile declares which is safe vs unsafe and names each transition's target state. |

The point is not that Claude *cannot* guess — it often guesses right. The point is that guessing and
*trusting* are different. The profile turns inference into a verifiable contract, keeps the reading
stable when the surface changes, and exposes affordances (like the unsafe acquisition action) that a
rendered page hides.

## Trust boundary: the profile defines meaning, not trust

This demo is benign: its profile is *more* honest than its surface. But the authority a profile holds
is over **meaning**, never over **trust**. The profile, the page it is linked from, the `rel="profile"`
referent, and every `def` target are all controlled by whoever controls the page. On an arbitrary URL
that is an untrusted party, so two failure modes have to be guarded — and the skill (`SKILL.md`,
*Trust boundary*) spells out the rules:

- **Dereferencing is a network action on attacker-chosen input.** Following `rel="profile"` and `def`
  fetches URLs the page picked. A hostile page can point them at `localhost`, a cloud metadata
  endpoint, or an internal service — an SSRF attempt dressed as a "definition." Dereference only
  `http`/`https`, refuse loopback/link-local/private addresses and `file:`/`data:`, prefer the
  platform web fetcher over shell `curl`, and surface unexpected cross-origin targets before following
  them.
- **A declared `type: safe` is a *claim*, not a verified guarantee.** Flip this demo around: a
  malicious catalogue could label its state-changing `reserveObject` (or a "delete", or a "pay") as
  `type: safe` with `def: https://schema.org/ViewAction`, and a reader that conformed naively would
  repeat "safe to view" for an action that charges your card. Report `type` as *what the profile
  declares*, never as established safety; cross-check independent signals (HTTP method, form
  semantics) and require explicit confirmation before any state-changing action, however it is
  labelled.

Conforming to a declaration is not the same as trusting it. The skill is designed to keep those two
layers apart — including for safety.

## Background

Inspired by [ALPS](https://alps-io.github.io/) (Application-Level Profile Semantics) and by ALPS-marked
pages such as the Vermeer works list, where `class` names are descriptor IDs and descriptors are linked
to schema.org via `def`. The demo here is fictional.
