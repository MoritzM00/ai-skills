# Self-Check (run before presenting)

Run the checks for the relevant tier. Flag, do not silently fix, anything that
looks like a factual, citation, or claim issue.

## All tiers

- [ ] Conforms to `style-guidelines.md`. Note any rule you could not verify.
- [ ] Uses American English spelling by default (e.g. "organize", not "organise") unless the user, venue, or surrounding document clearly requires another variant.
- [ ] Applies the colon rule from `style-guidelines.md` to revised prose.
- [ ] Avoids em dashes and en dashes in revised prose. Replace them with a sentence split, comma, or explicit connective.
- [ ] Avoids braided sentences: no sentence should chain several rhetorical jobs (e.g. contrast, rationale, method, dataset, and tradeoff) unless the length is technically necessary.
- [ ] If the detected user, venue, or document requirements call for **anonymization** (e.g. double-blind submission), no de-anonymizing content was introduced: no author names, identifying self-citations, or revealing links/acknowledgements.
- [ ] No invented results, citations, or quantitative claims.
- [ ] Notation and terminology match what the document already defines; nothing redefined or conflicting.
- [ ] Output is in the right medium (LaTeX by default), reusing existing macros/commands.
- [ ] No undefined symbols, undefined abbreviations, or dangling cross-references introduced.

## Review output

- [ ] Feedback stays in Review mode: no fixes, replacements, directives, rewritten sentences, paragraphs, or sections unless the user explicitly asked for revision.
- [ ] Scope, depth, and focus were either clear enough to infer or clarified before reviewing; a micro paragraph review did not ask unnecessary intake questions.
- [ ] Each finding has exactly one approved problem label from `review-vocabulary.md`, a severity tag, a concrete location, and a reason tied to a style principle, detected convention, context fact, or argument problem.
- [ ] The review does not claim a factual, statistical, or citation error unless the supplied material supports that judgment; otherwise it flags the point as something to verify.
- [ ] Context gathering stayed proportional to the tier; a micro review did not turn into a document-level audit.

## Section and above

- [ ] The piece fulfills its role in the argument (Step 2 context), not just locally correct prose.
- [ ] Opening and closing connect to adjacent sections; transitions are present.
- [ ] Length and density are in line with neighboring sections.
- [ ] Every non-trivial claim is either supported here or pointed to where it is supported.

## Chapter / document

- [ ] The narrative arc holds across the whole piece; each part advances the through-line.
- [ ] Terminology, notation, and the method/system name are consistent throughout.
- [ ] Forward/backward references resolve and are in the document's style.
- [ ] The claim→evidence map is complete: no claim is left unsupported, no result left unused.

## Anti-patterns to avoid

- Generic filler ("In recent years, … has attracted much attention") with no specific content.
- Hedging that hides whether a result holds, or overclaiming beyond the evidence.
- Restating the same point in successive sentences without adding information.
- Pulling in far more context than the scope required (a micro edit does not need the whole document).
