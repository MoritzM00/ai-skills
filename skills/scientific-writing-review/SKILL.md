---
name: scientific-writing-review
description: Use when reviewing (or, on request, revising) prose for a scientific or technical document (paper, thesis, technical report) — a sentence, a paragraph, a section, or a whole chapter/thesis — against scientific-writing conventions and the surrounding document's style. Default behavior is to REVIEW and give structured feedback, not to rewrite. Writing/editing only — assumes the research, results, and references already exist; does not run a research pipeline.
---

# Scientific Writing — Review & Revise

Review prose for scientific and technical documents — papers, theses, technical reports — across fields, and revise it only as an explicit second step. This skill is about *writing well*, not doing research: assume the ideas, results, and references already exist.

**This is primarily a reviewer.** The default deliverable is **structured feedback** on the text — what works, what doesn't, and why — at the level of a paragraph, a chapter, or a whole thesis. It does **not** rewrite the draft by default. Rewriting happens only in a separate, explicitly requested second step, and only on the parts the author chooses. This two-step separation is deliberate: the author keeps control of their own voice and decides which feedback to act on.

The defining move of this skill: **gather context in proportion to the scope under review.** Reviewing a single paragraph needs almost no surrounding context; reviewing a whole chapter or thesis needs the document's structure, terminology, and narrative arc. Spending the right amount of effort on context — no more, no less — is what separates a tailored review from generic notes.

This skill is **field- and venue-neutral**: it hardcodes no conventions. Citation style, length limits, tense/voice norms, anonymization, and all other conventions vary per document (a math thesis differs from a conference paper) and are detected from the document/template. Never assume a convention — detect it or ask. Use **American English** spelling by default (e.g. "organize", not "organise") unless the user, venue, or surrounding document clearly requires another variant. The universal writing principles it judges prose against live in `references/style-guidelines.md`.

This skill does *not* do literature search, citation discovery, or experiment design. If the request needs those, say so and stop. (The ARS `academic-paper` pipeline covers the full research-to-publication workflow.)

## The two modes

- **Review (default).** Set scope, depth, and focus → gather context → assess against only the relevant style guidelines and the review rubric → run the review self-check → deliver structured feedback. **Do not rewrite the draft.** End by offering to apply specific points.
- **Revise (only when asked).** After the author picks which feedback to act on — or explicitly asks for a rewrite up front — produce the revised text for *those* parts. Skip this entirely unless the author requests it.

Default to Review mode. Switch to Revise mode only on an explicit request such as "rewrite this", "apply points 2 and 4", or "now revise it".

The numbered steps below are the procedure: Steps 1–3 carry out a review; Step 4 is the revise mode and runs only when requested.

## Step 1 — Set scope, depth, and focus

Before reviewing, decide what kind of review the user wants. Do **not** ask reflexively: if the request, pasted text, or file context makes the intent clear, infer sensible defaults and proceed. Ask one short clarification question only when the scope, depth, or focus is ambiguous enough that it would materially change the review. When asking, state the default you would use if the user has no preference.

Classify the task into one tier:

| Tier | Covers | Typical requests |
|------|--------|------------------|
| **Micro** | a sentence or a single paragraph | "review this paragraph", "is this sentence clear?", "tighten feedback on this" |
| **Section** | one (sub)section | "review the related-work subsection", "critique the experimental setup", "check the contributions list" |
| **Chapter / thesis** | a full chapter or the whole document | "review the methods chapter", "give feedback on the introduction", "review my whole thesis" |

Then set review depth and focus:

- **Depth** — quick pass, standard review, or deep review. Default to standard review unless the user asks for a quick check or a deep critique.
- **Focus** — wording/style, sentence structure, local flow, argument/section structure, scientific precision, claim→evidence support, or consistency/conventions.
- **Micro default** — assume a targeted prose review focused on wording, sentence structure, local flow, clarity, and local terminology. Do not ask first for a normal single-paragraph review.
- **Section default** — assume a standard review of section role, organization, transitions, claims/evidence, terminology, and prose quality. Ask if the user may want only wording polish or only structural critique.
- **Chapter / thesis default** — ask for depth and focus before a full review unless the user already specified them. If proceeding without clarification, use a standard structural review focused on narrative arc, claim→evidence support, consistency, and high-impact prose issues.

## Step 2 — Gather context (proportional to scope)

Applies to both reviewing and revising. Read `references/context-checklist.md` for the full per-tier checklist. Read the surrounding files yourself when they are available; only ask the user for what you genuinely cannot find. Gather **exactly** what the tier calls for — pulling the whole document in to review one paragraph wastes effort and dilutes focus.

- **Micro** — the immediate neighbors (preceding and following sentence/paragraph) and the target style. Nothing more.
- **Section** — the section's role in the argument; the style and length of adjacent sections; terminology and notation already defined earlier; citation commands and conventions in use; what the reader is assumed to know at this point.
- **Chapter / thesis** — the outline / table of contents; the narrative arc and what each part must accomplish; cross-section terminology and notation consistency; the claim→evidence map; forward and backward references.

## Step 3 — Review (default deliverable)

This is the skill's main job. Read `references/review-rubric.md` (what to look for at each tier and how to structure the feedback). Use `references/style-guidelines.md` proportionally: for **Micro**, consult only the style sections needed for the issue at hand; for **Section** and **Chapter / thesis**, read the full style guide unless the task is narrowly bounded. Then produce **structured feedback** — do not rewrite the draft.

Let the **depth** and **focus** set in Step 1 shape the pass. A **quick** review surfaces only the few highest-severity findings; a **standard** review covers the rubric for the tier; a **deep** review works the rubric exhaustively. Restrict the rubric sections and style guidelines you apply to the chosen **focus** (e.g. wording only, or argument structure only) rather than reporting everything.

Each finding should: locate the issue (quote a short span or give the line/section), say what is wrong and why (tie it to a guideline or to the argument/context), and rate its severity. You may point toward a fix in a phrase, but do **not** supply full rewritten paragraphs here — that is Step 4.

Before presenting the review, run the **Review output** checks in `references/self-check.md`. End the review by listing the findings the author can choose from, and offer: "Tell me which of these to apply and I'll revise those parts."

## Step 4 — Revise (only when explicitly requested)

Skip this step unless the author asks for a rewrite (e.g. "apply points 2 and 4", "rewrite this paragraph", "now revise it"). Revise **only** the parts requested.

Produce **LaTeX** by default unless the surrounding file is clearly Markdown or plain prose. Match the existing document: reuse its macros, citation commands (`\cite`, `\citet`, `\citep`, …), math notation, environments, and sectioning. Do not introduce notation or terminology that conflicts with what is already defined (see Step 2).

As a house preference for text you generate, avoid introducing colons in revised prose — prefer a sentence split, comma, or explicit connective. Use a colon only when syntax or convention requires one (a LaTeX label/citation key, ratio, time, title/subtitle, code, or template field), or when the surrounding document clearly and consistently uses colons in that position. This governs your own output; it is not a defect to flag in the author's draft (the rubric flags only genuinely excessive colon use).

Preserve the author's meaning, claims, and voice — improve clarity, structure, and conformance to style, but never invent results, citations, or quantitative claims. Then run `references/self-check.md` for the relevant tier before presenting the revision, and flag (rather than silently fix) anything that looks like a factual or citation issue.
