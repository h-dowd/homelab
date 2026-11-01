---
name: homelab-change
description: How to change anything that runs in the homelab — which layer owns it (k8s/ via ArgoCD, bootstrap/, ansible/, lxc/, the host), the baselines every workload must meet (worker pin, securityContext, NetworkPolicy, namespace and prune guards, pinned images, limits), the manual-sync exceptions, validating before push, verifying after ArgoCD syncs, and rolling back through git. Use for any manifest, Helm values, VMRule, Alertmanager, Ansible, bootstrap or LXC change, adding a service, bumping a version, or any live `kubectl`/`qm`/`pct` change.
---

# Changing the homelab

Load `homelab-access` for the shells. `kubectl` runs on `ssh control-node`.

## 1. Find the layer that owns it

| Layer | Owns | Applied by | Self-heals? |
|---|---|---|---|
| `k8s/` | everything inside the cluster | ArgoCD, from `main`, within minutes of a push | yes. Live edits are reverted |
| `bootstrap/` | ArgoCD itself, its node pins, the `argocd-cm` and `argocd-rbac-cm` patches, Sealed Secrets install | `bootstrap.sh` once per rebuild, or the one step by hand | no |
| `ansible/` | the two VMs: journald, inotify, k3s `config.yaml`, kubelet shutdown, iSCSI ordering, the data disk | `ansible-playbook` by hand, `--check --diff` first | no |
| `lxc/` | files hand-installed in the LXCs | by hand, as the file's header says | no |
| the Proxmox host, VM sizes, Technitium records, Cloudflare, the router | — | by hand | no. Record it in `docs/` |

The change goes in the layer that owns the thing. Below `k8s/` nothing
re-applies itself, so drift is invisible until someone looks: a change made
by hand there has to land in the repo the same day.

## 2. Before you change it

- **Read the manifest's comments.** Constraints live next to what they
  constrain, and most odd-looking lines here have a dated comment explaining
  them and linking a lesson. Don't "clean up" a line whose comment you haven't
  read.
- `grep -rln '<component>' docs/lessons docs/reference/gotchas.md docs/adr`
  and read what turns up.
- Check the app is `Synced` and `Healthy` first
  (`kubectl -n argocd get applications`), so pre-existing drift isn't mistaken
  for yours afterwards.
- For a version bump: `docs/reference/resources.md` has the before-an-upgrade
  checklist and where each component's release notes live.

## 3. Baselines every workload meets

Each is an ADR. Read it before deviating, and a deviation needs a comment that
says why.

- **Pinned to the worker** (ADR 020). Every Deployment and StatefulSet sets
  `nodeSelector: kubernetes.io/hostname: k3s-worker1`. Control is tainted, so
  a missing pin shows up as Pending during a worker outage. With Helm charts,
  confirm the chart actually reads the key you set: the VictoriaMetrics stack
  ignored a top-level `nodeSelector` for months.
- **securityContext** (ADR 011). Pod `seccompProfile: RuntimeDefault`,
  container `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`,
  and only the capabilities the image's entrypoint needs added back. No
  blanket `runAsNonRoot`. Look at what the image does as root first.
- **NetworkPolicy** (ADR 011). Default-deny ingress on the namespaces listed in
  the ADR. Read `docs/lessons/k8s/netpol-fresh-pod-race.md` before adding one to a new
  namespace.
- **Its own `namespace.yaml`, no `CreateNamespace=true`** (ADR 014). Stateful
  namespaces and their irreplaceable PVCs carry `Prune=false`.
  `docs/runbooks/adding-a-service.md` step 8 still shows `CreateNamespace=true`.
  The ADR is the current rule.
- **Image tag pinned**, never `:latest`. **Requests and limits** set. An RWO
  PVC means `strategy.type: Recreate`. Consider `enableServiceLinks: false`.
- **Ingress exposure.** Any `*.henrydowd.dev` host on an Ingress is public the
  moment it syncs, because the tunnel carries a wildcard. LAN-only takes a
  `websecure` entrypoint pin plus `router.tls: "true"`. Authelia-gated takes a
  rule in Authelia's ConfigMap *before* the middleware annotation. Details:
  `adding-a-service.md` step 7.
- **A why-comment beside any non-default line**: the date, the cause and the
  lesson path, as in `k8s/apps/kiwix/deployment.yaml`.

New services follow `docs/runbooks/adding-a-service.md`, with the baselines
above applied on top.

## 4. The exceptions to "push and ArgoCD applies it"

- **`longhorn.yaml` and `sealed-secrets.yaml` are manual-sync** on purpose (see
  their headers). Committing them changes nothing live. Syncing them is
  Henry's call. Never "Sync All" in the ArgoCD UI.
- **ArgoCD is configured from `bootstrap/`**, not `k8s/`. A change to its pins or
  its ConfigMap patches is edited in `bootstrap/` and applied by hand, with
  Henry's OK.
- **Live-only state.** `kubectl taint` (k3s applies `node-taint` only at first
  registration), `qm set`, Technitium, Cloudflare, the router. Do it with
  Henry's OK, then record it where the next rebuild will find it.
- **Hand surgery on a managed object** needs selfHeal off first. Follow
  operations.md "Pausing autosync for manual surgery" *and* the `gotchas.md`
  ArgoCD entry on `root-app`, because disabling it on one app isn't enough.
  Turn it back on afterwards and say that you did.

## 5. Validate before asking to push

No `kubeconform`, `kubeseal` or `ansible-playbook` on the laptop. CI runs
kubeconform on push, which is after the fact. Before that:

```bash
# server-side dry run: real schemas, real CRDs, admission webhooks, changes nothing
ssh control-node 'kubectl apply --dry-run=server -f -' < k8s/apps/<app>/<file>.yaml
# what would change against the live object
ssh control-node 'kubectl diff -f -' < k8s/apps/<app>/<file>.yaml
```

For Helm values inline in an Application, the dry run only checks the
Application object. Bad indentation, a duplicate key or a misspelled key
becomes a silent `ComparisonError`, or a value that is ignored
(`docs/lessons/k8s/argocd-comparisonerror-silent-values.md`). Re-read the values block
against the chart's `values.yaml` for that version.

Secrets: build the Secret with `--dry-run=client -o yaml` piped straight into
`kubeseal` (operations.md, Sealed Secrets). The plaintext never touches a file,
the scratchpad or the chat.

Ansible: `--check --diff` first. Show Henry the diff, and ask before the real run.

## 6. Ask, commit, push, verify

1. Show Henry the diff and what it will do live. Ask.
2. Commit the change (`homelab-commit`), then ask before pushing.
3. After the push, watch it apply:
   ```bash
   ssh control-node 'kubectl -n argocd get app <app> -o jsonpath="{.status.sync.status} {.status.health.status} {.status.sync.revision}{\"\n\"}"'
   git rev-parse HEAD   # revision must match
   ssh control-node 'kubectl -n <ns> rollout status deploy/<name> --timeout=5m'
   ```
   Wait with one background `until` loop inside a single `ssh`. Don't poll by
   hand.
4. **Check that the thing works, not just the rollout.** Load the page, run the
   job, open the file. A 200 is not a smoke test
   (`docs/lessons/k8s/homepage-prerender-erofs.md`). For a public host, test from a Cloudflare
   edge with `--resolve`, not from the LAN.
5. Check that nothing went pending in vmalert (`homelab-alerts` §1) for the next
   few minutes.

**Rolling back** is `git revert` and a push, with Henry's OK. Not a `kubectl`
edit, which ArgoCD would undo anyway.

## 7. Record it

- The why-comment in the manifest (§3).
- Docs through `homelab-docs`: an ADR if alternatives were rejected,
  `services.md` for new config facts, `known-risks.md` if a risk moved, a lesson
  if it went wrong on the way.
- Commits split as in `homelab-commit` §4.
