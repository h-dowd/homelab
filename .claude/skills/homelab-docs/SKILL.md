---
name: homelab-docs
description: Where a piece of homelab knowledge goes and how to write it — lessons (new and recurrence), gotchas, known-risks updates, ADRs, runbooks, reference, plans and manifest comments; the template headings, index rows, the revision pass, and the checks before committing. Use whenever writing or editing anything under `docs/` or a README, after any incident, alert triage or change that taught something, or when Henry asks to "write it up" or "document it".
---

# Documenting the homelab

The canonical rules are in `docs/README.md`. Read it first if it isn't already
loaded. Where this skill seems to disagree with it, it wins and this skill
gets fixed.

## 1. Where it goes

| You have… | Write | Where | Also |
|---|---|---|---|
| a **new** failure mode: downtime, data risk, or over ~30 min of diagnosis | a lesson from `docs/lessons/TEMPLATE.md` | `docs/lessons/<domain>/<short-kebab-name>.md`. Domains: `infra`, `k8s`, `networking`, `storage`, `backup` | a row in `docs/lessons/README.md` **and** a one-paragraph entry in the right section of `docs/reference/gotchas.md` that links to it. All three, or it's a filing error |
| a **known** failure mode again, same cause confirmed | a `## Recurrence: YYYY-MM-DD` section at the end of the existing lesson | the existing lesson | update its `## Status` and its README row ("Recurred YYYY-MM-DD") |
| a known symptom with a **different** cause | a new lesson | as for a new failure mode | cross-link both under `## Related` |
| a decision with rejected alternatives | an ADR | `docs/adr/NNN-<kebab>.md` | a row in `docs/adr/README.md`. If it reverses an older ADR, mark that one `Superseded` with a pointer. Never rewrite it |
| a change to a risk's likelihood, cost or status | an `**Updated YYYY-MM-DD.**` paragraph in the item | `docs/reference/known-risks.md` | one dated sentence in the file's opening changelog paragraph, and the open-actions table if an item opens or closes (strike it through with `~~`, then `**done YYYY-MM-DD**`) |
| a procedure, done for the second time | a runbook, or a section of one | `docs/runbooks/` | a row in `docs/runbooks/README.md` |
| a command or fact you had to look up twice | a section or line | the matching `docs/reference/` file (ops commands go in `operations.md`) | a row in `docs/reference/README.md` if it's a new file |
| a pointer to upstream: chart, image, docs, release notes | a row | `docs/reference/resources.md` | — |
| scoped work that hasn't happened yet | a plan | `docs/plans/` | once executed, a line at the top saying so, with links to the ADR and lessons it produced |
| a constraint on one manifest line | a comment beside that line | the manifest | the doc links the file, and doesn't restate the comment |
| a rule for how Claude works here | a rule or a step | `CLAUDE.md` or the right skill | `claude:` commit |

Not everything earns a doc. A benign event that matched a lesson exactly and
needed no action gets nothing, unless it is the third or later time this
quarter. Then it gets a recurrence entry noting the frequency.

## 2. Formats

**Lesson.** Copy `docs/lessons/TEMPLATE.md` and keep its headings, in order:
Date · Time lost · Status · Context · Symptoms · Investigation · Root cause ·
Fix · Verification · Prevention · Related. Delete the template's HTML comment.
A recurrence section reuses the same headings as `###`, and stays short if
nothing new was learned. It still counts, because the recurrence count is what
prioritises prevention.

**Gotcha.** One paragraph under the right `##` section of `gotchas.md`. Bold
lead sentence that names the trap. Then the mechanism, and a link to the lesson.
Read the neighbouring entries first and match them.

**ADR.** Take the next free number (`ls docs/adr`; numbers are never reused,
and 001–003 and 016 are not free). Header:

```markdown
# ADR NNN: <the decision, as a statement>

**Status:** Accepted
**Date:** YYYY-MM-DD
```

Then `## Problem`, `## Decision`, `## Rejected`, `## Consequences`, the shape of
ADRs 019 and 020. Add `## Applied` or `## Verification` when the change has
gone in. The README row is a one-paragraph summary, including what was rejected.

**known-risks.md item update.** Don't rewrite the item's history. Append the
dated paragraph, and correct figures in place only where the paragraph says
what changed.

**Runbook.** Numbered steps, each with the command that actually works here.
Say what success looks like after each step that can fail.

## 3. Style

- **Absolute dates (`YYYY-MM-DD`), times in UTC.** For `Europe/Dublin`
  schedules, give both: `03:00 Europe/Dublin = 02:00 UTC (IST)`. Never
  "yesterday", "last night" or "recently".
- **Evidence over narrative.** Real log lines in ` ```text ` blocks, trimmed to
  the line that shows the cause. Real pod, Job, alert and node names. The
  commands that actually ran, not the ones that should have.
- **Say how you know.** "Inferred from …, not from logs" where that's true.
  Keep the dead ends. They are often the most useful part.
- **Root cause is a mechanism**, not "it broke". **Prevention takes a
  position**: a specific change, or "accepted risk" and the reason.
- **No version numbers.** Point at the manifest:
  `grep -rn targetRevision k8s/`. Image tags in a log line quoted as evidence
  are fine.
- **Link, don't restate.** If another doc already says it, link that section.
  Anything written twice drifts.
- **Henry's voice stays.** Many docs are first person ("I picked…"). Edit them
  in that voice, and don't convert them to a neutral one.
- **Keep the existing structure.** In a long reference file, add to the section
  that exists. Don't add a new top-level section for one fact.

## 4. The revision pass

Run `docs/reference/human-register-revision-guide.md` §12 over every new or
substantially rewritten lesson, ADR, plan, gotcha or runbook section before
committing. Read §11 first. These are technical docs, so bullets, tables,
uniform structure and repeated key terms stay. Numbers, names, commands, alert
names and log lines are never reworded for style. Stop when the next change
would only make the text different, not better.

`CLAUDE.md` and the skills are written for a model, so the whole guide is
suspended for them (§11, last row).

## 5. Before committing

```bash
# every path the new text links or mentions exists
grep -oE '(docs|k8s|ansible|bootstrap|lxc)/[A-Za-z0-9_./-]+' <file> | sed 's/[.,]$//' | sort -u | while read p; do test -e "$p" || echo "MISSING $p"; done
# every new file has its index row
git status --short docs/ | grep '^??'
# no relative dates slipped in
grep -nEi 'yesterday|today|last night|this morning|recently' <file>
```

- A new lesson has its `lessons/README.md` row and its `gotchas.md` entry.
- Any `known-risks.md` change has its changelog sentence.
- If the work was an alert triage and the lesson covers an alert that will
  fire again, the pattern table in `homelab-alerts` §3 gets a row.
- `README.md` at the top level is the public map. Touch it only when a new
  top-level doc or directory needs a link.

Then commit with `homelab-commit`: `docs: <what> -- <why>`.
