# Know Your Unknowns Pattern Reference

Use this reference after `SKILL.md` selects a pattern. Keep the artifact tied to real code, data, diff, design, or user-supplied goals.

## 1. Blindspot Pass

Use before editing unfamiliar risky code: auth, billing, permissions, migrations, storage, data lifecycle, concurrency, security, deployment.

Artifact:

- "What you probably think" vs "what the code/history shows".
- Blindspot cards with code path, history, why it matters, and prompt constraint.
- A rewritten implementation prompt that bakes in the findings.

Blog-faithful mechanics:

- Show several concrete blindspot cards, not a single risk summary.
- Each card should include a copyable prompt fix.
- Assemble the card fixes into one improved implementation prompt.

Avoid:

- Generic "add tests" advice.
- Risks with no path, commit, migration, issue, or local evidence.

## 2. Teach Me My Unknowns

Use when the user has intent but lacks domain vocabulary or judgment criteria.

Artifact:

- Mental model of the domain.
- Vocabulary ladder from beginner terms to useful expert terms.
- Examples of bad/good phrasing.
- A better prompt the user can reuse.
- Optional interactive controls when parameters are visual or experiential.

Blog-faithful mechanics:

- Turn a vague request into professional vocabulary.
- Include a ladder from basic words to domain terms.
- Include a live or simulated comparison when the domain is visual.

Avoid:

- Encyclopedia summaries that do not improve the next prompt.

## 3. Four Design Directions

Use when design taste, density, information architecture, or product mood is unclear.

Artifact:

- 3-4 intentionally different directions using the same data.
- For each direction: best fit, tradeoff, steal/skip controls or bullets.
- A synthesized design prompt using selected pieces.

Keep variables controlled: same content, different design philosophy.

Blog-faithful mechanics:

- The directions should be intentionally incompatible.
- Let the user mark what to steal and what to skip.
- Generate a reply from those reactions.

## 4. Mock Before You Wire

Use before wiring a real app when layout, interaction, or control model is uncertain.

Artifact:

- Static mock with realistic density and states.
- Switchable layout/control alternatives.
- High-leverage questions whose answers would change structure.
- Reply template: selected option, changes requested, unresolved questions.

Do not connect real data unless necessary.

Blog-faithful mechanics:

- Emphasize that no real app code has been touched.
- Provide several toggleable placements or interaction variants.
- Include A/B questions about structural choices.

## 5. Brainstorm the Intervention

Use when the problem is broad: churn, activation, quality, reliability, cost, support burden, adoption.

Artifact:

- 8-12 interventions ordered from quick ship to major project.
- Each item includes code/data evidence, expected impact, cost, risk, and what it proves.
- A way for the user to mark "resonates", "maybe", "skip".

Avoid pure product brainstorming without codebase or evidence grounding.

Blog-faithful mechanics:

- Plot ideas from quick intervention to larger bet.
- Make each intervention trace back to something that already exists in the product/code.
- Assemble selected interventions into a next-step reply.

## 6. The Interview

Use when requirements are ambiguous and guessing would affect architecture.

Artifact:

- Ask one question at a time.
- Order by blast radius: architecture, data model, permissions, API contract, UX flow, policy, polish.
- Explain why each answer matters.
- Offer 2-4 concrete options plus "other".
- End with a decision table and generated implementation prompt.

Do not ask a long unordered list of questions.

Blog-faithful mechanics:

- Show progress through the interview.
- Sort questions by architectural blast radius.
- Separate architecture/data questions from UX/policy/polish questions.

## 7. Point At A Reference

Use when porting, reimplementing, or matching a reference implementation.

Artifact:

- Semantic summary of reference behavior.
- Side-by-side source/target snippets when useful.
- Preserved exactly / deliberately changed / dropped table.
- Edge-case table with expected behavior.
- Sign-off prompt focused on semantics, not diff shape.

Watch for language traps: integer division, inclusive ranges, monotonic clocks, async boundaries, retries, ordering, idempotency, cancellation, precision, timezone, nullability.

Blog-faithful mechanics:

- Make the agent prove it understood the reference before implementation.
- Include matched excerpts or pseudocode only where needed.
- Make edge cases reviewable before code is written.

## 8. The Tweakable Plan

Use when the user wants a plan but likely needs to modify assumptions.

Artifact:

- Start with effort, touched files, risk, migration/deploy notes.
- Section A: decisions the user probably wants to tweak.
- Section B: sequencing.
- Section C: mechanical work.
- Each tweakable decision includes default, alternatives, cost, and when to choose differently.

Do not bury architecture choices in execution order.

Blog-faithful mechanics:

- Sort by likelihood of user modification, not execution order.
- Flag schema/type/user-facing choices first.
- Collapse trusted mechanical work at the bottom.

## 9. Implementation Notes

Use during long or uncertain implementation.

Artifact:

- Running log with timestamp or step number.
- Categories: plan-confirmed, discovery, deviation, conservative choice, human todo.
- For deviations: original plan, code reality, chosen conservative path, revisit condition.
- End with fold-back bullets for the next plan/prompt.

This is not a changelog. It preserves why the implementation diverged.

Blog-faithful mechanics:

- Capture surprises as they happen, not only after the build.
- For each deviation, include the conservative call made.
- End with a short set of bullets to fold into attempt two.

## 10. The Buy-in Doc

Use when code/prototype/spec is ready but needs approval.

Artifact:

- Lead with demo or concrete user-facing result.
- Pitch: problem, evidence, what changed, why now.
- Reviewer objections as Q/A.
- Spec at a glance.
- Risk, rollback, blast radius, known gaps.
- Named asks: who must approve what.

Write for decision-makers, not just code reviewers.

Blog-faithful mechanics:

- Package prototype, spec, and implementation notes into one skimmable doc.
- Lead with demo.
- Pre-answer reviewer objections with evidence.
- Name each approver and their exact sign-off area.

## 11. Quiz Me Before I Merge

Use before merging risky changes.

Artifact:

- Mental model diagram or step-by-step behavior.
- Non-obvious behaviors and why they are deliberate.
- Existing assumptions the change depends on.
- Quiz questions mapped to incident/review decisions.
- Wrong answers point back to the relevant section.
- Final checklist: CI, migration, release note, deploy watch, rollback, known follow-ups.

The quiz should test operational understanding, not trivia.

Blog-faithful mechanics:

- Make the report cover context, intuition, and what changed.
- Put a quiz at the bottom that the user must pass.
- Wrong answers should point back to the exact report section.
