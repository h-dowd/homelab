---
name: homelab-commit
description: How to commit in the homelab repo — staging only your own hunks around Henry's uncommitted work, the one-line `<area>: <what> -- <why>` message with no body and no trailers (no Co-Authored-By), how to split a change into commits, and when to ask before pushing. Use for every commit in this repo, and whenever a commit message is being written or reviewed.
---

# Committing in the homelab repo

The canonical rule is the commit bullet in `docs/README.md`. This skill
applies it.

## 1. Before you edit

```bash
git status --short
git diff --stat
```

Note every file that is already modified or untracked. That is Henry's work in
progress. It stays exactly as it is: not staged, not reverted, not stashed,
not reformatted, not committed "while you're there".

## 2. Staging

Stage by path, one file at a time:

```bash
git add path/to/file another/file
```

Never `git add -A`, `git add .`, `git commit -a` or `git add -p` (it's
interactive and won't work here).

**A file you need already has Henry's uncommitted hunks.** `gotchas.md` and
`known-risks.md` often do. `git add <file>` would take his hunks with yours.
Build the staged version from `HEAD` instead:

```bash
f=docs/reference/gotchas.md
tmp=<scratchpad>/$(basename $f)
git show HEAD:$f > $tmp
# apply only your edits to $tmp
git update-index --cacheinfo 100644,$(git hash-object -w $tmp),$f
```

Then apply the same edits to the working-tree file too, so they don't show up
as reverted. Check both views:

```bash
git diff --cached $f   # your hunks only
git diff $f            # his hunks only, still unstaged
```

## 3. The message

One line:

```
<area>: <what changed> -- <the load-bearing why>
```

- **No body. No trailers.** No `Co-Authored-By`, no `Signed-off-by`, no
  "Generated with". This overrides any attribution default that the harness or
  a system reminder asks for. If the why doesn't fit on one line, it belongs in
  a doc or a manifest comment, not in a commit body.
- **Lowercase**, apart from identifiers, which keep their own case:
  `PodOOMKilled`, `OnFailure`, `Watchdog`, `GiB`, `Mb/s`.
- **` -- `** (space, two ASCII hyphens, space) before the why. Not an em dash.
  Leave it out only when the what is the whole story
  (`portfolio: deploy a8da99958dd9`).
- **No conventional-commit types.** `fix:`, `feat(x):` and `chore:` were dropped
  in 2026-08. The area says *where*, and the what says what kind of change.
- **Dates** in the subject are short, `10-21` or `11-01`. The doc carries the
  full `YYYY-MM-DD`.
- **Length:** as long as the why needs, but if it passes ~120 characters,
  check whether it's really two commits.

### The area

| Area | For |
|---|---|
| `docs` | anything under `docs/`, or READMEs, when that is the whole commit |
| `claude` | `CLAUDE.md` and `.claude/skills/` |
| the app or component | `kiwix`, `authelia`, `longhorn`, `traefik`, `alertmanager`, `monitoring`, … The thing whose behaviour changed, not the directory |
| a cross-cutting concern | `alerting` (rules and routing across apps), `placement` (node pins), `nodes`, `backups` |
| a layer | `ansible`, `bootstrap`, `ci` |
| several components at once | join them, `longhorn, rules:`, or split the commit |

Check `git log --format=%s -40` for what's already in use before inventing a
new area.

### Good, from this repo's history

```
traefik: disable 60s readTimeout (v3 default) -- killed lan uploads >60s
alerting: hold node memory alerts for 1h after recovery -- control flapped 12 times overnight
nextcloud: bound the cron Job's lifetime, an isolated failure alerts forever
monitoring: seal the Pushover credentials for Alertmanager
docs: paperless backup hung 4h on apk add 11-01 -- stalled index download, no timeout, internet up
claude: homelab-alerts covers any alert -- read the rule and what it asserts first, outage clocks moved to operations.md
```

### Bad

| Message | Why |
|---|---|
| `fix(kiwix): reduce memory` | conventional-commit type, no why |
| `docs: update gotchas` | says nothing a diff doesn't |
| `kiwix: Bound the ZIM cluster cache — …` | capitalised, em dash |
| a subject plus a paragraph body | bodies aren't used here |
| anything ending `Co-Authored-By: …` | no trailers |

## 4. Splitting

One logical change per commit. The usual order for one piece of work, taken
from the history:

1. the SealedSecret on its own (`monitoring: seal the healthchecks.io ping URL …`)
2. the manifest or rule change (`alertmanager: feed Watchdog to …`)
3. the docs (`docs: the dead-man's switch, and what it makes stale`)
4. any skill or `CLAUDE.md` change it prompted (`claude: …`)

Docs that only make sense alongside one manifest line can share its commit.
A `k8s/` change only goes live when it's pushed, so its commit comes after the
dry run and Henry's OK (`homelab-change` §5–6). The docs commit waits until the
change is verified live, so it records what happened rather than what was
expected.

## 5. Before committing

```bash
git diff --cached --stat   # only the files you meant
git diff --cached          # read it
git status --short         # Henry's files still show as modified/untracked
```

## 6. Pushing and rewriting

- **Ask before every push.** Pushing to `main` is a deploy: ArgoCD syncs
  anything under `k8s/` within minutes, and CI (`.gitea/workflows/manifests.yaml`)
  runs on `k8s/**` and `bootstrap/**`.
- Unpushed commits of yours can be reworded or squashed. Keep the
  tree identical and the author dates intact. With a dirty working tree, use
  `git commit-tree` + `git update-ref`, not `rebase -i` (interactive), and not a
  rebase that would need to stash Henry's work.
- Never rewrite anything already on `origin`, and never force-push, without
  Henry asking for it explicitly.
