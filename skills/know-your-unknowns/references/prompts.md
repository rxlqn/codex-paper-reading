# Copyable Prompts

Replace bracketed placeholders before use.

For normal AI coding use, do not force HTML. Ask for the smallest artifact that exposes the unknown:

```text
Choose the lightest useful artifact format for this task: Markdown, table, checklist, diagram, interview, implementation log, quiz, or self-contained HTML only if interaction/visual comparison matters.
```

For a blog-faithful HTML artifact, prepend this line to any pattern prompt:

```text
Create a single self-contained HTML artifact with a prompt card at the top, a "What the agent produced" section, and a concrete next-step output.
```

## Blindspot Pass

```text
I need to implement [task] in [module/codebase], but I do not know this area well.
Before writing code, do a blindspot pass.

Inspect the relevant code paths, tests, migrations, config, and recent history if available.
Find unknown unknowns that would cause a naive implementation to be wrong.

For each blindspot, include:
- what I might assume;
- what the code/history actually shows;
- why it matters;
- files or evidence to inspect;
- the prompt constraint that should be added before implementation.

End with a revised implementation prompt. Do not write code yet.
```

## Teach Me My Unknowns

```text
I want [goal], but I lack the domain vocabulary to ask for it well.
Teach me the unknowns I need before asking an agent to do the work.

Give me:
- a compact mental model;
- a vocabulary ladder from basic to useful expert terms;
- what good vs bad output looks like;
- the parameters I should be able to specify;
- a rewritten prompt using the right terms.
```

## Four Design Directions

```text
Use the same realistic content and produce four clearly different design directions for [screen/feature].

For each direction, show:
- what design philosophy it represents;
- who it is best for;
- what tradeoff it makes;
- 3 elements I might steal;
- 3 reasons I might skip it.

End with a synthesis prompt that combines the selected direction and stolen details.
```

## Mock Before You Wire

```text
Before touching the real app, create a throwaway static HTML mock for [UI/interaction].

Use realistic fake data and include the important states.
Show [2-4] layout or interaction alternatives if relevant.
Include the high-leverage questions whose answers would change the implementation structure.
End with a reply template I can fill in to choose a direction.
```

## Brainstorm The Intervention

```text
We have this problem: [problem/metric/user pain].
Do not jump to one solution.

Inspect the code/product surface and brainstorm 8-12 possible interventions ordered from "can ship soon" to "major project".

For each intervention, include:
- code/data evidence that makes it plausible;
- expected impact;
- implementation cost;
- risk or dependency;
- what we would learn if we shipped it.
```

## The Interview

```text
Interview me one question at a time about [feature].
Prioritize questions where my answer would change architecture, data model, permissions, API contract, or user flow.

For each question:
- explain why the answer matters;
- offer concrete options and tradeoffs;
- let me answer in my own wording.

After the interview, produce:
- a decision table;
- fixed constraints for implementation;
- a ready-to-use implementation prompt.
```

## Point At A Reference

```text
I want to implement [target] based on this reference: [reference path/link].
Before coding, create a semantics map.

Include:
- the reference behavior in plain language;
- preserved exactly;
- deliberately changed;
- dropped or out of scope;
- edge cases and expected behavior;
- language/runtime traps that could change semantics.

Do not write code until the semantic contract is clear.
```

## The Tweakable Plan

```text
Create an implementation plan for [task], but order it by likelihood that I may want to tweak it, not by execution order.

Start with effort, files touched, risk, migration/deploy notes.
Then separate:
- decisions I probably want to change;
- recommended sequencing;
- mechanical work you can handle without debate.

For each tweakable decision, include default, alternatives, cost, and when to choose differently.
```

## Implementation Notes

```text
While implementing [task], maintain implementation notes.

Whenever code reality forces a discovery or deviation, record:
- original plan;
- code reality;
- conservative choice made;
- revisit condition or human decision needed.

At the end, fold the discoveries back into a revised plan or next prompt.
```

## The Buy-in Doc

```text
Package [prototype/spec/implementation notes/diff] into a buy-in doc for [stakeholders].
Lead with the demo or user-facing result.

Include:
- why this matters now;
- what changed;
- likely reviewer objections with answers;
- spec at a glance;
- risk, rollback, blast radius, known gaps;
- named asks: who needs to approve what.
```

## Quiz Me Before I Merge

```text
I want to make sure I understand [diff/change] before merge/release.
Create a merge-readiness report with a quiz I must pass.

Include:
- mental model of old vs new behavior;
- non-obvious behaviors and why they are deliberate;
- existing assumptions this change depends on;
- operational questions I would need to answer during an incident;
- feedback that maps wrong answers back to the report section;
- final merge/release checklist.
```
