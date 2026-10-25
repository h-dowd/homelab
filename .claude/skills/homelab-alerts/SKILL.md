---
name: homelab-alerts
description: Triage, fix and write up firing homelab alerts — list what Alertmanager/vmalert has firing, match each alert to a known lesson before diagnosing, remediate within the limits below, verify the alert set is back to baseline, then file the documentation the repo's rules require (lesson, recurrence, gotcha, known-risk). Use whenever Henry mentions alerts, alert emails or Pushover notifications, failed/missing backups, `KubeJobFailed`, `BackupJobFailed`, `PodCrashLooping` or any other alert name, or asks to "check the alerts".
---

# Homelab alert triage

Access first: load the `homelab-access` skill. `kubectl` runs on
`ssh control-node`, never on the laptop. If SSH stalls right after
`Server accepts key`, or fails with `agent refused operation`, the 1Password
agent is waiting for Henry to approve or unlock. Say so and stop. Don't retry in
a loop. When he says to prompt again, make one foreground attempt with a
generous timeout so the approval dialog has time to appear. Run several
commands per `ssh` call: each connection can need its own approval. Off the home LAN, the route in is
the `homelab` NetworkManager WireGuard profile (`nmcli connection up homelab`).
While it is up, the no-`pct enter` rule for LXC 101/102 applies.

## 1. List what is firing

```bash
ssh control-node '
P=vmalertmanager-vm-victoria-metrics-k8s-stack-0
kubectl -n monitoring exec $P -c alertmanager -- \
  amtool --alertmanager.url=http://localhost:9093 alert query'
# -o extended dumps every annotation: tens of KB for a dozen alerts. Don't.
```

**Baseline (always firing, by design):** `Watchdog`, `KubeCPUOvercommit`,
`KubeMemoryOvercommit`. Anything beyond those three is the work. Done means
the list is back to exactly those three, or Henry has agreed to leave
something firing.

Alertmanager's state is short-lived (emptyDir, resets on pod restart). For
*when* something fired, or what fired while you weren't looking, query the
stored `ALERTS` series in vmsingle instead. The commands are in
`docs/reference/operations.md`, Alerting: vmsingle's ClusterIP, `curl` from
`control-node`.

Notifications arrive late when the internet is down: email, Pushover and the
healthchecks.io deadman all need external DNS. An alert's arrival time is not
when it fired.

## 2. Match before you diagnose

Most alerts here have fired before. Before you investigate, grep for the alert
name:

```bash
grep -rln '<AlertName>' docs/lessons docs/reference/gotchas.md
```

Known patterns, so you can recognise them on sight:

| Alert(s) | Usual cause | Lesson |
|---|---|---|
| `BackupJobFailed` + `KubeJobFailed` + `BackupJobMissing`, all four namespaces, ~02:00–03:40 UTC | ISP outage over the backup window. The Jobs die ~10.5 / 1 / 12 / 10.5 min after start (nextcloud / gitea / immich / paperless). Seen 2026-10-20, 21, 22 | `backup/internet-outage-backups-failed.md` |
| `BackupJobFailed` / `KubeJobFailed` that won't clear | Failed Job object retained forever (count-triggered history, no TTL on backups) | `k8s/kubejobfailed-isolated-failure-never-evicted.md` |
| `KubeJobFailed` on `nextcloud-cron` | Ran during a node/control-plane restart. TTL clears it in 24h | same |
| `PodCrashLooping` cluster ~20 min after a worker reboot, no `PodOOMKilled` | Longhorn reattach, benign | `k8s/worker-reboot-alert-storm.md` |
| `NodeMemoryLowControl` hours after a reboot | bloated control-node `longhorn-manager` | same |
| `LonghornVolumeDegraded` / faulted volumes | disk pressure blocking auto-salvage, or shutdown ordering | `storage/longhorn-autosalvage-blocked-diskpressure.md`, `storage/planned-reboot-longhorn-killed-under-writes.md` |
| `PodOOMKilled` | check the pod's lesson (kiwix, file-parser, act-runner, grafana) | `k8s/*oom*` |
| `PortfolioMetricsAbsent` after a portfolio restart | lazy gauge, one real page load clears it | gotchas.md, Monitoring |

A match is a hypothesis, not a diagnosis. Confirm it with this incident's
evidence before applying the old fix. The 2026-09-19 recurrence looked like
the 09-02 one and had a different cause.

## 3. Diagnose

- **Cluster health first:** `kubectl get nodes`, non-Running pods outside
  completed Jobs, Deployments ready. That tells you whether this is one
  workload or the platform.
- **Then the specific evidence:** Job conditions, pod logs, events, and
  `kube_job_status_failed` / other series in vmsingle for exact times.
- **Backup Jobs leave no logs** (`restartPolicy: OnFailure`, the pod is deleted
  at `backoffLimit`). For outages, bracket the window from independent clocks:
  - failed Alertmanager notifies. The container log rotates after ~3 days, so
    use the stored `alertmanager_notifications_failed_total{integration="webhook"}`
    series (operations.md, Alerting). It is the sharpest clock, ±10 min.
  - the end of each outage: a router restart shows as a ~2 min double
    `enp0s31f6: NIC Link is Down/Up` bounce in `ssh pve 'journalctl -k'`,
    followed by a new WAN IP in
    `ssh pve 'pct exec 101 -- cat /var/log/cloudflare-ddns.log'`
    (never `pct enter`). A re-IP *without* a NIC bounce or notify failures is
    not an outage.
  - chrony on `pve` (`journalctl -u chronyd`), `Can't synchronise`. It lags the
    start by hours.
- **Confirm the cause is over** before re-running anything. For B2:
  `curl -sS -o /dev/null -w '%{http_code}\n' https://s3.eu-central-003.backblazeb2.com/`
  from `control-node` should return `403` quickly.
- Write down dead ends as you go. They go in the doc.

## 4. Remediate

**Do without asking**: these are the documented fixes, and reversible:
- Re-run a failed backup or cron Job, **unless a later scheduled run in that
  namespace has already completed**. Then the backup has recovered, and
  re-running only adds a snapshot (and Gitea downtime). Check the logs of the
  latest `Complete` Job for `snapshot … saved`. Otherwise re-run:
  `kubectl -n <ns> create job --from=cronjob/<ns>-backup <ns>-backup-manual-$(date +%s)`.
  The gitea backup scales Gitea to 0 for ~1.5 min. immich takes the longest (~15 min).
- Once the re-run is `Complete`, delete the failed Jobs that are keeping
  `BackupJobFailed` / `KubeJobFailed` up, **only** if their pods are already gone
  (`kubectl -n <ns> get pods -l job-name=<job>` is empty). If logs still exist,
  save them to the scratchpad first.
- Bounce a stateless pod a lesson says to bounce (for example `longhorn-manager`).

**Ask Henry first:**
- any silence (it hides the next real alert, and is lost on Alertmanager restart)
- any manifest, VMRule or Alertmanager config change (it's GitOps, so it means
  a commit, which ArgoCD then applies)
- deleting or restarting anything that holds state or still has logs
- node, VM or Proxmox host reboots, Longhorn replica surgery, anything in LXC 101
- muting or disabling a rule. Prove the component is healthy first
  (`k3s-control-plane-false-positives.md`)

Prevention work a lesson lists as "proposed, not yet applied" is Henry's call.
Point out that it recurred and what that costs. Don't implement it unasked.

## 5. Verify

Re-run step 1. The firing list should be back to the three baseline alerts.
After a Job delete, kube-state-metrics drops the series at once, but vmsingle
keeps returning the last sample for ~5 min, so the alert takes 5–10 min to
resolve. Wait with one background `until` loop inside a single `ssh`; don't
poll. For backups, also confirm each manual Job is `Complete` and that
its log ends with a snapshot dated today, and that Gitea is back
(`kubectl -n gitea rollout status deploy/gitea`).
`BackupJobMissing` clears by itself once a backup succeeds.
ArgoCD's CronJob `Degraded` only clears on a successful *scheduled-or-manual*
run, and deleting Jobs does not clear it.

## 6. Document

The canonical rules are in `docs/README.md`. Read it if anything below seems
to conflict, and follow it. This section applies those rules to alert work.

### When, and where

| What the triage found | Write | Where |
|---|---|---|
| A **new** failure mode: downtime, data risk, or over ~30 min of diagnosis | A new lesson from `docs/lessons/TEMPLATE.md` | `docs/lessons/<domain>/<short-kebab-name>.md`, plus a row in `docs/lessons/README.md`, plus a one-paragraph entry in `docs/reference/gotchas.md` under the right section linking to it. All three, or it's a filing error |
| A **known** failure mode recurring, cause confirmed the same | A `## Recurrence: YYYY-MM-DD` section appended to the existing lesson: what fired, the window, what (if anything) differed, the commands run. Keep it short if nothing new was learned. It still counts, because the recurrence count is what prioritises the prevention work | the existing lesson. Update its `## Status` line and its `docs/lessons/README.md` summary (add "Recurred YYYY-MM-DD") |
| Known alert, **different** cause | A new lesson, cross-linked both ways under `## Related` | as for a new failure mode |
| The incident changes a risk's likelihood, cost or status | An `**Updated YYYY-MM-DD.**` paragraph in the item, and the dated changelog sentence at the top of the file | `docs/reference/known-risks.md` |
| You ran a multi-step procedure for the second time | A runbook, or a section in an existing one | `docs/runbooks/` plus its README row |
| A command you had to work out that the next triage will need | Add it to the relevant section | `docs/reference/operations.md` |
| A manifest or rule fix | A comment beside the changed lines saying why. The lesson links the commit | the manifest |
| A pure baseline/benign alert that matched a lesson exactly and needed no action | Nothing, unless it is the 3rd+ time this quarter, in which case a recurrence entry noting the frequency | — |

Also check whether *this skill's* known-patterns table needs a row. If you
wrote a new lesson for an alert that will fire again, it does.

### Style

- **Follow the template's headings, in order.** Date · Time lost · Status ·
  Context · Symptoms · Investigation · Root cause · Fix · Verification ·
  Prevention · Related. Recurrence sections reuse the same sub-headings at `###`.
- **Run the revision pass** in `docs/reference/human-register-revision-guide.md`
  §12 before committing. These are technical docs, so the §11 exemptions apply:
  bullets, tables and repeated key terms stay. Numbers, names, commands, alert
  names and log lines are never reworded for style.
- **Absolute dates (`YYYY-MM-DD`), times in UTC.** When a schedule is in
  `Europe/Dublin`, give both (`03:00 Europe/Dublin = 02:00 UTC (IST)`). Never
  "yesterday" or "last night".
- **Evidence over narrative:** real log lines in ` ```text ` blocks, trimmed to
  the smoking gun. Real alert names, Job names, timestamps, and the commands
  that actually ran, with their output.
- **Say how you know.** If something is inferred and not observed, write "inferred
  from …, not from logs". The investigation section includes the dead ends.
- **Root cause is a mechanism**, not "it broke". Prevention takes a position:
  a specific change, or "accepted risk", with the reason.
- **No version numbers in docs.** Point at the manifest (`grep -rn targetRevision k8s/`).
- Don't restate what another doc already says. Link it.

### Committing

Commit only the files this triage touched, staged by path. The working tree
often holds Henry's unrelated in-progress work, so never use `git add -A` or
`git commit -a`. Run `git status` / `git diff` before you edit. If a file you
need already has his uncommitted hunks (gotchas.md and known-risks.md often
do), you can't use `git add <file>`, and `git add -p` is interactive. Stage
your hunks alone instead: apply the same edits to `git show HEAD:<file>` in the
scratchpad, then
`git update-index --cacheinfo 100644,$(git hash-object -w <tmp>),<file>`.
Check with `git diff --cached`, and confirm `git diff` still shows his hunks. Messages follow `docs/README.md`: terse lowercase one-liner,
`docs: <what> -- <load-bearing why>`, for example
`docs: backup outage recurrence 10-25 -- isp drop over backup window again`.
Commit once the docs are written and verified. Ask before pushing.
