---
name: semantic-web-fetch
description: >
  Interpret a web page in conformance with the profile it declares about itself.
  Use this skill whenever a page carries a rel="profile" link declaration, or when the request
  asks for a reading that conforms to a profile or vocabulary such as ALPS / microformats / RDFa /
  microdata / schema.org. Trigger it for requests like "read it as raw HTML plus its profile,"
  "interpret it conformant to the profile," "read it as ALPS," "bind it to the definitions,"
  or "determine the meaning without filling gaps by inference." Trigger it as well when a URL is
  given and the question concerns its semantic structure, descriptors, transitions, or affordances,
  and when class names map to descriptor IDs or descriptors are linked to external vocabularies via
  `def`. The default behavior of merely reading extracted text loses the authority of meaning and
  the transition information that this skill recovers, so engage it whenever a profile is in evidence.
---

# Semantic Web Fetch

A skill for interpreting a page **in conformance with** the profile it declares. This is not a
procedure manual; it is the definition of where the authority over meaning resides. The procedure
follows from the principles, so prioritize honoring the principles.

## Why this is needed

By default, when given a URL I read extracted text (Markdown-equivalent). That is usually enough to
understand the data by contextual inference — but such understanding is probabilistic guesswork, not
the meaning the author declared. Worse, the extracted form structurally loses affordance information:
which state a link transitions to. A profile supplies both of these — the authority of meaning and
the transitions — as a machine-readable contract. The `rel="profile"` declaration is itself the
contract that says "this page conforms to this definition," and this skill puts that contract into
effect.

## The three principles (the definitions to uphold)

### 1. Authority
Locate the authority over meaning in the profile the document declares. Do not make my own
contextual inference the authority. Rather than guessing "this is probably the creation year,"
receive the fact that a declared descriptor *defines* it as the creation year.

### 2. Binding
Treat only elements that can be bound to declared descriptors as grounds for meaning. Match HTML
elements (typically the `class` attribute, or the respective attributes of microformats / RDFa /
microdata) to descriptor IDs in the profile. When a descriptor carries a `def` (a reference to an
external vocabulary such as schema.org), include that referent in the grounds for meaning as well.

### 3. No inferred bindings
Do not invent bindings absent from the definition. For elements the profile does not mention, do not
fill in meaning by inference and treat them as part of a descriptor. Report any element that could
not be bound honestly as "unbound." Never conflate what was filled in by inference with what was
determined from the definition. This is the line that separates reading from conforming-to-definition.

## Behavior that follows from the principles

To satisfy the principles above, the following behavior results (this is a consequence of the
principles, not the principles themselves).

- Fetch the **raw HTML source**, not extracted text. If tags and attributes are dropped, the binding
  targets disappear. In the Claude.ai environment the source can be fetched directly with `curl` in
  `bash`.
- When a `rel="profile"` link declaration is detected, **also fetch its referent (the profile
  itself)** and reconcile the two. The HTML alone does not reveal the descriptor definitions, their
  `def`, or their transitions (`rt`).
- Establish the element-to-descriptor correspondence and follow each descriptor's `def` to fix its
  authority.
- Handle transitions (affordances). For ALPS, read `type` (safe / idempotent / unsafe) and `rt` (the
  target state of the transition), and treat "which operation leads to which state" as meaning.

Where to find the relevant vocabulary: for ALPS, the JSON/XML body pointed to by `rel="profile"`;
for microformats, the `class` attribute (h-card, etc.); for RDFa, `vocab` / `typeof` / `property`;
for microdata, `itemscope` / `itemtype` / `itemprop`; for schema.org, the URL referenced by `def` or
`itemtype`.

## How to report

When conveying the interpretation, distinguish the following.

- **Declared structure**: which profile is conformed to, and which descriptors are defined.
- **Established bindings**: which element is bound to which descriptor, and what the `def` referent is.
- **Transitions**: which operation leads to which state (type and target).
- **Unbound**: elements with no correspondence in the definition. Do not fill these in by inference.

The data content itself (the contents of a table, etc.) may be summarized separately if needed, but
make clear that this is "what I read" and sits at a different layer from "the meaning the definition
guarantees."

## The line

"Whether I can infer the meaning" and "whether that meaning may be trusted" are separate questions.
This skill addresses the latter. Where inference suffices, the benefit is slight. But it tells when
you want meaning treated as a verifiable contract, when you want a footing that does not break as the
structure changes, and when you want to follow transitions as a client rather than merely read.
A profile is a mechanism premised on the idea that machines cannot infer meaning; yet even now that
inference is possible, its role of turning inference into certainty remains.
