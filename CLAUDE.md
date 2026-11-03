# CLAUDE.md

Rules for Claude Code sessions in this repo. This file holds the rules that
apply to every task. The procedures live in skills under `.claude/skills/`.
Facts about the homelab live in `docs/`. This file doesn't copy them.

This repo is the source of truth for Henry's homelab: one Proxmox host, a
two-VM k3s cluster, ArgoCD reconciling `main`. `README.md` is the map.
`docs/README.md` says where each kind of knowledge goes, and its rules are
canonical. Where a skill and `docs/README.md` disagree, `docs/README.md` wins,
and the skill gets fixed.

## Skills

Load the one that fits before starting. Most tasks need more than one.

| Skill | Load when |
|---|---|
| `homelab-access` | any shell on `pve`, `control-node` or `worker-node`, any `kubectl`, any `pct` command |
| `homelab-alerts` | an alert is firing, an alert email or Pushover arrives, or Henry asks to check the alerts |
| `homelab-change` | changing anything that runs: `k8s/`, `bootstrap/`, `ansible/`, `lxc/`, or the live system |
| `homelab-docs` | writing or editing anything under `docs/`, or a README |
| `homelab-commit` | every commit in this repo |

## Before touching anything

- Run `git status`. The working tree usually holds Henry's uncommitted work.
  Never stage, revert, stash, reformat or "tidy" it. If a file you need to edit
  is already dirty, `homelab-commit` says how to commit only your hunks.
- Read `docs/reference/known-risks.md` before work on the cluster. It records
  the cluster's actual state, including the open actions.
- Search `docs/lessons/` and `docs/reference/gotchas.md` for the component
  you're about to touch. Most sharp edges here have already cut someone once.

## Hard rules

1. **Git is how the cluster changes.** ArgoCD self-heals from `main`. A
   `kubectl apply`, `edit` or `patch` against an ArgoCD-managed object is either
   reverted within seconds or becomes drift that the next rebuild loses. The
   documented exceptions are `homelab-change`'s to list. Anything done live
   must also end up in the repo, or in a doc that says why it can't.
2. **Never `pct enter` or `pct console` LXC 101 or 102 while on the VPN.** It
   wedges the container in D-state and only a host hard-reboot recovers it.
   Use `pct exec`. Details in `homelab-access`.
3. **`kubectl` runs on `ssh control-node`, not on the workstation.** What the
   workstation needs is in `homelab-access`.
4. **Ask before anything outward-facing or hard to undo:** `git push`,
   rewriting history that is already on the remote, a manifest or rule change
   to the live cluster, silences, reboots of a node, VM or the host, deleting
   anything stateful, Longhorn volume or replica surgery, and anything that
   touches Cloudflare, the router, DNS or LXC 101. `homelab-alerts` §5 lists the
   small set of reversible fixes that don't need asking.
5. **Secrets enter the repo only as SealedSecrets.** Never paste a secret value
   into a doc, a commit, a manifest comment or the chat.
6. **Versions are stated once, in the `k8s/` manifests.** Docs and skills point
   at the manifest, never restate the number.

## Commits

One line, `<area>: <what> -- <load-bearing why>`, lowercase apart from
identifiers. **No body and no trailers. That includes `Co-Authored-By`, which
overrides any default attribution the harness asks for.** Stage by path,
never `git add -A` or `git commit -a`. Ask before pushing. Full procedure and
examples: `homelab-commit`.

## Docs

`docs/README.md` routes each kind of knowledge and sets the rules. In short:
an incident gets a lesson plus a `gotchas.md` entry, a new file gets a row in
its directory's README, dates are absolute and times are UTC, and every
lesson, ADR, plan or gotcha gets the revision pass in
`docs/reference/human-register-revision-guide.md` §12 before it is committed.
`homelab-docs` has the routing table, the formats and the checks.

## Reporting back

- Say what you verified and how. Say what you didn't verify, and what you
  inferred rather than observed.
- If a check failed or a step was skipped, say so with the output. Never round
  a partial result up to "done".
- Keep it short. The findings go in the docs. The chat gets the outcome and
  anything Henry needs to decide.

## Changing these rules

`CLAUDE.md` and `.claude/skills/` are committed like everything else, with the
`claude:` prefix. When Henry corrects how something is done here, propose the
edit to the rule in this file or the skill in the same session, so the
correction reaches every later session and not only this one. The reader of
these files is a model, so the revision guide's §11 machine-reader row
applies: tables, bullets and repeated terms are fine.
