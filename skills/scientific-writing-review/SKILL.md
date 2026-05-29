---
name: scientific-writing-review
description: Use when reviewing (or, on request, revising) prose for a scientific or technical document (paper, thesis, technical report) — a sentence, a paragraph, a section, or a whole chapter/thesis — against scientific-writing conventions and the surrounding document's style. Default behaviour is to REVIEW and give structured feedback, not to rewrite. Writing/editing only: assumes the research, results, and references already exist; does not run a research pipeline.
---

# Scientific Writing — Review & Revise

Review prose for scientific and technical documents — papers, theses, technical reports — across fields, and revise it only as an explicit second step. This skill is about *writing well*, not doing research: assume the ideas, results, and references already exist.

**This is primarily a reviewer.** The default deliverable is **structured feedback** on the text — what works, what doesn't, and why — at the level of a paragraph, a chapter, or a whole thesis. It does **not** rewrite the draft by default. Rewriting happens only in a separate, explicitly requested second step, and only on the parts the author chooses. This two-step separation is deliberate: the author keeps control of their own voice and decides which feedback to act on.

The defining move of this skill: **gather context in proportion to the scope under review.** Reviewing a single paragraph needs almost no surrounding context; reviewing a whole chapter or thesis needs the document's structure, terminology, and narrative arc. Spending the right amount of effort on context — no more, no less — is what separates a tailored review from generic notes.

This skill is **field- and venue-neutral**: it hardcodes no conventions. Citation style, length limits, tense/voice norms, anonymisation, and all other conventions vary per document (a math thesis differs from a conference paper) and are detected from the document/template. Never assume a convention — detect it or ask. The universal writing principles it judges prose against live in `references/style-guidelines.md`.

This skill does *not* do literature search, citation discovery, or experiment design. If the request needs those, say so and stop. (The ARS `academic-paper` pipeline covers the full research-to-publication workflow.)

## The two-step process

1. **Review (default).** Detect scope → gather context → assess against the style guidelines and the review rubric → deliver structured feedback. **Do not rewrite the draft.** End by offering to apply specific points.
2. **Revise (only when asked).** After the author picks which feedback to act on — or explicitly asks for a rewrite up front — produce the revised text for *those* parts. Skip this step entirely unless the author requests it.

Default to Step 1. Move to Step 2 only on an explicit request such as "rewrite this", "apply points 2 and 4", or "now revise it".

## Step 1 — Determine scope

Classify the task into one tier. If it is genuinely ambiguous, ask the user; otherwise infer from the request.

| Tier | Covers | Typical requests |
|------|--------|------------------|
| **Micro** | a sentence or a single paragraph | "review this paragraph", "is this sentence clear?", "tighten feedback on this" |
| **Section** | one (sub)section | "review the related-work subsection", "critique the experimental setup", "check the contributions list" |
| **Chapter / thesis** | a full chapter or the whole document | "review the methods chapter", "give feedback on the introduction", "review my whole thesis" |

## Step 2 — Gather context (proportional to scope)

Applies to both reviewing and revising. Read `references/context-checklist.md` for the full per-tier checklist. Read the surrounding files yourself when they are available; only ask the user for what you genuinely cannot find. Gather **exactly** what the tier calls for — pulling the whole document in to review one paragraph wastes effort and dilutes focus.

- **Micro** — the immediate neighbours (preceding and following sentence/paragraph) and the target style. Nothing more.
- **Section** — the section's role in the argument; the style and length of adjacent sections; terminology and notation already defined earlier; citation commands and conventions in use; what the reader is assumed to know at this point.
- **Chapter / thesis** — the outline / table of contents; the narrative arc and what each part must accomplish; cross-section terminology and notation consistency; the claim→evidence map; forward and backward references.

## Step 3 — Review (default deliverable)

This is the skill's main job. Read `references/style-guidelines.md` (the universal writing principles to judge against) and `references/review-rubric.md` (what to look for at each tier and how to structure the feedback). Then produce **structured feedback** — do not rewrite the draft.

Each finding should: locate the issue (quote a short span or give the line/section), say what is wrong and why (tie it to a guideline or to the argument/context), and rate its severity. You may point toward a fix in a phrase, but do **not** supply full rewritten paragraphs here — that is Step 4.

End the review by listing the findings the author can choose from, and offer: "Tell me which of these to apply and I'll revise those parts."

## Step 4 — Revise (only when explicitly requested)

Skip this step unless the author asks for a rewrite (e.g. "apply points 2 and 4", "rewrite this paragraph", "now revise it"). Revise **only** the parts requested.

Produce **LaTeX** by default unless the surrounding file is clearly Markdown or plain prose. Match the existing document: reuse its macros, citation commands (`\cite`, `\citet`, `\citep`, …), math notation, environments, and sectioning. Do not introduce notation or terminology that conflicts with what is already defined (see Step 2).

Preserve the author's meaning, claims, and voice — improve clarity, structure, and conformance to style, but never invent results, citations, or quantitative claims. Then run `references/self-check.md` for the relevant tier before presenting the revision, and flag (rather than silently fix) anything that looks like a factual or citation issue.
