---
name: know-your-unknowns
description: Use this skill when an AI coding or agent workflow task has meaningful uncertainty and should surface unknowns before coding, during implementation, or before shipping. Triggers include requests for blindspot passes, clarification interviews, design directions, throwaway mocks or HTML artifacts, intervention brainstorming, reference-port semantics, tweakable implementation plans, implementation notes, buy-in docs, merge readiness reports, or quizzes that verify understanding before merge/release.
---

# Know Your Unknowns

Use this skill to choose and produce an artifact that turns hidden uncertainty into explicit decisions for AI-assisted engineering work. The default output is the smallest useful artifact: often a Markdown table, checklist, interview, implementation log, quiz, or plan. Create self-contained HTML only when interaction, visual comparison, clicking, progressive disclosure, or quiz feedback materially improves the decision.

This skill is a reusable workflow adaptation of the `Know your unknowns` examples from the HTML-effectiveness gallery. It should reproduce the blog's decision mechanics, not overfit to HTML as the only output format, clone the original visual design, or copy large source text.

## Core Rule

Do not treat "unknowns" as generic risks. Tie every unknown to one of:

- a decision the user must make;
- a codebase fact that must be verified;
- a semantic boundary that must be preserved;
- a reviewer/stakeholder objection that must be answered;
- a future incident question someone must answer correctly.

## Stage Router

| Stage | Use when | Choose |
|---|---|---|
| Pre-implementation | The task is not yet implemented, or requirements/design/semantics are unclear | patterns 1-8 |
| During implementation | Work is underway and code reality is changing the plan | pattern 9 |
| Post-implementation | The change is done or near done and needs buy-in, review, merge, or release confidence | patterns 10-11 |

## Pattern Menu

| # | Pattern | Best for | Required output |
|---:|---|---|---|
| 1 | Blindspot pass | unfamiliar risky code areas | blindspots with code evidence and a better implementation prompt |
| 2 | Teach me my unknowns | user lacks domain vocabulary | vocabulary ladder, judgment criteria, improved prompt language |
| 3 | Four design directions | visual/product direction is unclear | 3-4 intentionally different options and a steal/skip decision surface |
| 4 | Mock before you wire | UI/interaction shape is uncertain | static mock or lightweight prototype, high-leverage questions, reply template |
| 5 | Brainstorm the intervention | problem is broad but intervention space is unclear | code-grounded options ordered by cost/ambition |
| 6 | The interview | ambiguous requirements could change architecture | one-question-at-a-time interview ordered by blast radius |
| 7 | Point at a reference | porting or reimplementing a reference | semantics map: preserved, changed, dropped, edge cases |
| 8 | The tweakable plan | implementation plan contains human decisions | plan ordered by likelihood of user modification |
| 9 | Implementation notes | long implementation may deviate from plan | running log of discoveries, deviations, conservative choices, revisit items |
| 10 | The buy-in doc | change needs stakeholder sign-off | demo-first pitch, objections, spec summary, risk/rollback, named asks |
| 11 | Quiz me before I merge | reviewer must prove operational understanding | merge-readiness report plus quiz mapped to incident decisions |

For full details, read `references/patterns.md`. For source alignment against the original examples, read `references/source-map.md`. For copyable prompts, read `references/prompts.md`.

## Artifact Format Rule

Choose the least heavy format that exposes the unknown:

- Use Markdown for durable reasoning, plans, decision tables, checklists, source maps, and implementation notes.
- Use diagrams when relationships, flows, or dependencies are the unknown.
- Use an interactive HTML page when the user needs to compare visual options, click through states, answer an interview, or get quiz feedback.
- Do not generate HTML just because the source blog used HTML.

## Blog-faithful Artifact Shell

When explicitly asked to create a blog-style HTML artifact, preserve this structure:

1. Page header with stage and pattern title.
2. Prompt card showing the prompt being answered.
3. Divider such as "What the agent produced".
4. The artifact itself: interactive comparison, interview, mock, report, notes, or quiz.
5. A concrete next-step output: generated prompt, decision table, reply template, sign-off request, or merge checklist.

Use the original stage counts and ordering: Pre-implementation has patterns 1-8, During implementation has pattern 9, Post-implementation has patterns 10-11.

## Workflow

1. Identify the stage: pre, during, or post.
2. Pick one primary pattern. Combine patterns only when the task clearly needs it.
3. Ground the artifact in the real source material: code, diff, issue, design, data, reference implementation, or user-provided goal.
4. State what unknown the artifact is designed to expose.
5. Choose the artifact format: Markdown by default; diagram when relationships matter; self-contained HTML when interaction, comparison, clicking, quiz feedback, or visual judgment would materially improve the decision.
6. End with the next prompt, decision table, sign-off request, or checklist that moves the work forward.

## Output Standards

- Prefer concrete evidence over generic advice.
- Make choices comparable: same data, different assumptions.
- Separate architecture/data decisions from UX/policy/polish decisions.
- Preserve user edits and existing repo conventions.
- If generating HTML, keep it disposable, static, accessible, and safe to open locally.
- Do not force HTML for ordinary coding tasks; the workflow is artifact-first, not HTML-first.
- If the user is preparing public material, keep third-party source content linked and summarized rather than copied wholesale.
- If reproducing the blog workflow for publication, cite the source page and use original wording sparingly; prefer paraphrased prompts and your own example content.

## Fast Selection Heuristics

- If the user says "I don't know this codebase": use Blindspot pass.
- If the user says "make it more professional" but lacks terms: use Teach me my unknowns.
- If the user asks for UI direction: use Four design directions or Mock before you wire.
- If the user asks "what should we do about this metric/problem?": use Brainstorm the intervention.
- If answers may change data model, permissions, API, or flow: use The interview.
- If there is a source implementation to match: use Point at a reference.
- If the user asks for a plan and may want to tweak it: use The tweakable plan.
- If implementation is already running: use Implementation notes.
- If the goal is approval: use The buy-in doc.
- If the goal is merge confidence: use Quiz me before I merge.
