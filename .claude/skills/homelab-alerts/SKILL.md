---
name: homelab-alerts
description: Triage, fix and write up any firing homelab alert (memory, workloads, reboots, storage, backups, Authelia, portfolio, or the chart's default Kube* rules). Read the rule, work out what its condition asserts and whether it still holds, match it against past lessons, find the cause, remediate within the limits below, verify, then file the documentation the repo's rules require (new lesson, recurrence, gotcha, known-risk, runbook). Use whenever Henry mentions alerts, an alert email or Pushover notification, an alert name (`PodOOMKilled`, `LonghornVolumeDegraded`, `KubeJobFailed`, …), or asks to "check the alerts".
---

# Homelab alert triage

## 0. Access

Load the `homelab-access` skill. `kubectl` runs on `ssh control-node`, never on
the laptop. Batch several commands into one `ssh` call. Each connection can
need its own 1Password approval.

- **SSH stalls after `Server accepts key`, or fails with `agent refused
  operation`:** the 1Password agent is waiting for Henry to approve or unlock.
  Say so and stop. Don't retry in a loop. When he says to prompt again, make one
  foreground attempt with a generous timeout so the dialog has time to appear.
- **Off the home LAN:** connect the `homelab` NetworkManager WireGuard profile
  with `nmcli connection up homelab`, and say you did. While it is up, the
  no-`pct enter` rule for LXC 101/102 applies.

## 1. List what is firing, and what is about to

Run on `control-node`:

```bash
# vmalert: firing + pending, with each rule's expression
VMA=$(kubectl -n monitoring get svc vmalert-vm-victoria-metrics-k8s-stack -o jsonpath='{.spec.clusterIP}')
curl -s http://$VMA:8080/api/v1/alerts | python3 -c '
import json,sys
for a in json.load(sys.stdin)["data"]["alerts"]:
    print(a["state"], a["name"], a["labels"].get("namespace",""), "|", a["expression"][:100].replace("\n"," "))'

# Alertmanager: what is being notified, and any silences
P=vmalertmanager-vm-victoria-metrics-k8s-stack-0
AM="kubectl -n monitoring exec $P -c alertmanager -- amtool --alertmanager.url=http://localhost:9093"
$AM alert query          # not -o extended: tens of KB of annotations
$AM silence query
```

**Baseline (always firing, by design):** `Watchdog`, `KubeCPUOvercommit` and
`KubeMemoryOvercommit`. Anything else is the work. Done means only the baseline
is left, or Henry has agreed to leave something firing.

How alerts reach Henry: criticals go to Pushover and email, warnings to email,
and `Watchdog` goes to the healthchecks.io deadman. Routes and inhibitions are in
`k8s/infrastructure/victoria-metrics.yaml`. Alertmanager's state lives in an
emptyDir and resets when the pod restarts. For history (when an alert started,
or what fired and already resolved) query the stored `ALERTS` series in
vmsingle (`docs/reference/operations.md`, Alerting). Delivery needs the
internet, so an alert's arrival time is not when it fired.

## 2. Read the rule before you read the system

Find the rule. Custom rules are in `k8s/apps/monitoring/homelab-rules.yaml`
(`grep -n -A15 'alert: <Name>'`). Read its comments too, because they often
record why it is shaped the way it is. Chart rules (`Kube*`, `Node*`, `Target*`, …)
come from `defaultRules` in `k8s/infrastructure/victoria-metrics.yaml`, and
vmalert's output above already shows their expressions.

Then work out **what the expression asserts**. That decides what "fixed"
means:

| Shape | Examples | It clears when… | Trap |
|---|---|---|---|
| A state object exists | `KubeJobFailed`, `BackupJobFailed`, `DeploymentReplicasMismatch` | the object changes or is deleted | fixing the cause doesn't clear it. A failed Job is kept until deleted |
| Rate over a window | `PodCrashLooping` (`increase[15m] > 3`) | the window slides past | can fire after recovery, e.g. a stale storm after a reboot |
| Level / threshold | `NodeMemoryLow*`, `DiskFillingUp`, `LonghornProvisioningHeadroomLow` | capacity actually changes | raising a limit just to silence it moves the problem out a day |
| Time since last success | `BackupJobMissing`, `NextcloudCronStale` | the next success | the cause may be long over |
| Absence | `*MetricsAbsent`, `BackupJobsAbsent` | the series reappears | the pipeline may be fine and the series just lazily created (portfolio) |
| Something happened | `NodeRebooted`, `ProxmoxHostRestarted`, `PodOOMKilled` | by itself, after the window | the alert is a pointer. The question is *why* it happened |

**Run the expression in vmsingle now.** If it returns nothing, the condition
is over and you are looking at a stale alert, a long `for:`, or a resolve still
in flight.

## 3. Match against what has happened before

```bash
grep -rln '<AlertName>' docs/lessons docs/reference/gotchas.md docs/reference/known-risks.md
```

Read `docs/lessons/README.md` for the one-line summaries. Patterns worth
recognising on sight:

| Alert(s) | Usual cause | Lesson |
|---|---|---|
| Several unrelated alerts in one burst | one shared event: a reboot, an internet outage, Longhorn, or the control plane gone | see step 4 |
| `PodCrashLooping` ~20 min after a worker reboot, no `PodOOMKilled` | Longhorn reattach, benign, self-resolves | `k8s/worker-reboot-alert-storm.md` |
| `NodeMemoryLowControl` hours after a reboot | bloated control-node `longhorn-manager`, bounce it | same |
| `PodOOMKilled` | per-app: kiwix cache, file-parser page count, act-runner GC, grafana | `k8s/*oom*` |
| `LonghornVolumeDegraded` / `LonghornDiskUnschedulable` | disk pressure blocking auto-salvage, or shutdown ordering | `storage/longhorn-autosalvage-blocked-diskpressure.md`, `storage/planned-reboot-longhorn-killed-under-writes.md` |
| `KubeJobFailed` / `BackupJobFailed` that won't clear | failed Job retained forever (count-triggered history, and backups have no TTL) | `k8s/kubejobfailed-isolated-failure-never-evicted.md` |
| `BackupJob*` in all namespaces, 02:00–03:40 UTC | internet down over the backup window | `backup/internet-outage-backups-failed.md` |
| `BackupJobFailed` in one namespace, `DeadlineExceeded`, internet up | `apk add restic` stalled mid-download with no timeout. Raw rx bytes flat under ~3 MB | `backup/apk-add-hang-no-timeout.md` |
| Job `DeadlineExceeded` long after its deadline | control plane was absent, not a hung job | gotchas.md, Monitoring |
| `PortfolioMetricsAbsent` after a portfolio restart | lazy gauge, one real page load clears it | gotchas.md, Monitoring |
| `KubeContainerWaiting` / `DeploymentReplicasMismatch` with `ImagePullBackOff`, `403` on `ghcr.io/token` | the image owner moved (GitHub account rename) or the package went private. Running pods hide it until the next pod creation | `k8s/ghcr-owner-rename-imagepull-403.md` |
| `NodeRebooted` / `ProxmoxHostRestarted` while `PlannedMaintenance` fires | `ansible/upgrade.yml` rebooting on purpose. Muted by design | `ansible/README.md`, Package upgrades |
| Control-plane `*Down` (scheduler, etcd, …) | k3s isn't kubeadm. Those rules are disabled, so if one appears, the config drifted | `k8s/k3s-control-plane-false-positives.md` |

A match is a hypothesis, not a diagnosis. Confirm it from this incident's
evidence before applying the old fix. The 2026-09-19 `KubeJobFailed` looked
like the 09-02 one and had a different cause.

## 4. Diagnose

**Scope first.** `kubectl get nodes`, then the pods that aren't Running or
Completed (`kubectl get pods -A | grep -vE 'Running|Completed'`), then
`kubectl -n argocd get applications` for anything not Synced/Healthy. One
workload, or the platform?

**Then the timeline.** When did each alert start? Use the `ALERTS` series.
Several alerts starting together point to a shared cause, so check these
before investigating any single alert:
- **a reboot.** `journalctl --list-boots` on the node or `pve`. A journal that
  ends mid-line is an unclean stop. Check `NodeRebooted` / `ProxmoxHostRestarted`
  history.
- **the internet.** Alertmanager logs `lookup … server misbehaving` for
  Brevo, Pushover and hc-ping. Bracket the outage as in operations.md
  (Alerting: notify-failure series, NIC bounce + ddns, chrony).
- **storage.** `kubectl -n longhorn-system get volumes.longhorn.io`, plus disk
  conditions on `nodes.longhorn.io`.
- **the control plane.** Job and lease timestamps that jump all at once.

**Evidence by area:**

| Area | Look at |
|---|---|
| Pods / workloads | `kubectl describe pod` (Last State, Reason, Events), `logs --previous`, `get events -n <ns> --sort-by=.lastTimestamp` |
| Memory | `kubectl top`, `container_memory_working_set_bytes` over time (a sawtooth means it will recur), cgroup `rss` vs `cache` split (kiwix lesson) |
| Nodes / host | `ssh worker-node` / `ssh pve`: `journalctl -b -1 -e`, `free`, `df`. On pve also `zfs list`, the `lvs` thin pool, `qm status` |
| Storage | Longhorn volume / replica / engine CRs, `df /mnt/longhorn` on the worker, the PVC's events |
| Jobs / backups | Job conditions and their timestamps (`kubectl get job -o yaml`), whether pods survive (backup pods often don't: `OnFailure`), `kube_job_status_*` series, `snapshot … saved` in the logs |
| Ingress / auth | `curl -H 'Host: <h>' http://192.168.1.200/ -I` (operations.md "When something looks wrong"), Authelia pod logs |
| GitOps drift | `kubectl -n argocd get app <a> -o yaml`, which shows sync status and `ComparisonError`. The live spec may not be the repo's (`argocd-comparisonerror-silent-values.md`) |

**Confirm the cause is over** before re-running or restarting anything.
Note dead ends as you go, because they go in the doc.

## 5. Remediate

**Do without asking** (documented fixes, reversible):
- anything read-only
- re-run a failed Job with `kubectl -n <ns> create job --from=cronjob/<cj> <cj>-manual-$(date +%s)`,
  unless a later scheduled run has already completed (check its logs). A
  backup re-run scales Gitea to 0 for ~1.5 min. immich takes ~15 min.
- delete a failed Job that is keeping a state-object alert up, once the cause
  is fixed and **only** if its pods are gone (`kubectl -n <ns> get pods -l job-name=<job>`).
  If logs survive, save them to the scratchpad first.
- restart a stateless pod that a lesson says to restart (for example
  `longhorn-manager` on the control node)
- trigger a series that is created lazily (one real portfolio page load)

**Ask Henry first:**
- a silence. It hides the next real alert, and it is lost when Alertmanager restarts.
- any manifest, VMRule, values or Alertmanager change. It's GitOps, so it means
  a commit that ArgoCD applies.
- raising a memory, CPU or disk limit. Find out why usage grew first, and offer
  the fix for that.
- restarting or deleting anything stateful, a PVC, or anything that still has logs
- node, VM or Proxmox host reboots, Longhorn replica or volume surgery, anything
  in LXC 101, the router, Cloudflare or DNS
- muting or disabling a rule. Prove the component is healthy first
  (`k3s-control-plane-false-positives.md`).

Prevention that a lesson marks "proposed, not yet applied" is Henry's call.
Tell him it recurred and what that cost. Don't implement it unasked.

## 6. Verify

- The rule's expression returns nothing in vmsingle, vmalert shows it inactive,
  and `amtool alert query` is back to the baseline.
- **The thing works, not just the alert.** Load the page, open a file, check the
  Job's log for its success line. A 200 is not a smoke test
  (`k8s/homepage-prerender-erofs.md`).
- Resolution lags. After a deleted object or a recovered pod, vmsingle keeps
  returning the last sample for ~5 min, so expect 5–10 min before Alertmanager
  resolves. Wait with one background `until` loop inside a single `ssh`. Don't
  poll by hand.
- Some things don't clear the way you'd expect. ArgoCD's CronJob `Degraded`
  only clears after a successful run. Deleting Jobs or a hard refresh doesn't
  clear it.

## 7. Document

Load `homelab-docs`. It has the routing table, the lesson template, the style
rules and the revision pass. The cases alert work produces:

| What the triage found | Write |
|---|---|
| a **new** failure mode: downtime, data risk, or over ~30 min of diagnosis | a new lesson, plus its `lessons/README.md` row and `gotchas.md` entry |
| a **known** failure mode recurring, same cause confirmed | a `## Recurrence: YYYY-MM-DD` section on the existing lesson: what fired, the window, what differed, the commands run |
| a known alert with a **different** cause | a new lesson, cross-linked both ways |
| an alert that is **wrong** (false positive, wrong threshold, wrong scope) | the rule fix through `homelab-change` (ask first), with a comment beside it saying why, and a lesson if it cost real time |
| a change to a risk's likelihood, cost or status | the `known-risks.md` update |
| a command you had to work out that the next triage will need | a line in `docs/reference/operations.md`, Alerting |
| a benign alert that matched a lesson exactly and needed no action | nothing, unless it's the 3rd+ time this quarter. Then a recurrence entry noting the frequency |

Then check whether step 3's table in *this skill* needs a row. If you wrote a
new lesson for an alert that will fire again, it does.

Commit through `homelab-commit`: only the files this triage touched, staged by
path around Henry's uncommitted work, with a message like
`docs: kiwix oom recurrence 11-02 -- bot crawl again, cache bound holding`.
Commit once the docs are written and the alert is verified clear. Ask before
pushing.
