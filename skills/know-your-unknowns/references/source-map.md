# Source Alignment Map

This skill is adapted from the `Know your unknowns` examples in the HTML-effectiveness gallery:

- Main page: https://thariqs.github.io/html-effectiveness/unknowns/
- Parent gallery: https://thariqs.github.io/html-effectiveness/
- Upstream repository: https://github.com/ThariqS/html-effectiveness
- Upstream commit checked: `1787245d94aa680edf18b52027e3f859032776ba`
- Upstream license: Apache License 2.0

Use this map when checking whether a generated artifact still follows the original blog structure. It is intentionally paraphrased for shareability; load the source pages if exact prompts or visual details are required.

## Global Structure

| Source structure | Skill equivalent |
|---|---|
| 11 self-contained HTML artifacts | 11 reusable patterns |
| Exact prompt at top | Prompt card in artifact shell |
| Produced artifact below prompt | Artifact body after divider |
| Before / during / after implementation | Pre / During / Post stage router |
| HTML used to expose unknowns | Any artifact format chosen by interaction and decision need |

## Pattern Source Map

| # | Stage | Source page | Source intent | Reproduction requirement |
|---:|---|---|---|---|
| 1 | Pre | `01-blindspot-pass.html` | Find unknown unknowns in unfamiliar auth code before implementation | Produce concrete blindspot cards and fold them into a better implementation prompt |
| 2 | Pre | `02-color-grading-explainer.html` | Teach domain vocabulary before doing unfamiliar work | Produce mental model, vocabulary ladder, judgment criteria, and improved prompt language |
| 3 | Pre | `03-design-directions.html` | Render four incompatible design directions so reaction can guide taste | Use the same content across distinct directions and collect steal/skip reactions |
| 4 | Pre | `04-toolbar-mock.html` | Create a throwaway toolbar mock before touching app code | Provide fake-data UI, toggleable placements, structural questions, and reply template |
| 5 | Pre | `05-churn-brainstorm.html` | Explore intervention space for churn using the actual codebase | Order interventions by cost/ambition and ground each one in existing product/code evidence |
| 6 | Pre | `06-interview.html` | Interview the user one question at a time, ordered by architecture impact | Ask sequential questions, show blast radius, and output decisions plus implementation prompt |
| 7 | Pre | `07-reference-port.html` | Prove understanding of a reference implementation before porting | Produce semantics map, gotchas, preserved/changed/dropped table, and edge cases |
| 8 | Pre | `08-implementation-plan.html` | Sort plan by likely user edits, not execution order | Lead with tweakable data/type/UX decisions and push mechanical work down |
| 9 | During | `09-implementation-notes.html` | Log mid-build deviations so surprises feed the next attempt | Maintain running discoveries/deviations with conservative choices and fold-back bullets |
| 10 | Post | `10-pitch-doc.html` | Package prototype/spec/notes into a Slack-ready buy-in doc | Lead with demo, answer objections, summarize spec, risk, rollback, and named sign-offs |
| 11 | Post | `11-change-quiz.html` | Verify merge understanding with a report and required quiz | Explain old/new behavior, non-obvious decisions, dependencies, quiz questions, and checklist |

## Review Checklist

Use this checklist before publishing a derived artifact:

- Does it preserve the correct stage and ordering?
- Does each pattern expose a specific unknown rather than generic risk?
- Does the output include a next-step object: prompt, decision table, reply, sign-off, or checklist?
- If HTML is generated, is there a specific reason Markdown/table/checklist would be weaker?
- If blog-style HTML is requested, does it include prompt card, produced artifact, and interaction or feedback?
- Is the source linked, and are large source passages avoided?
