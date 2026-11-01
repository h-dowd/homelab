# Incident: an ISP outage over the backup window failed all four nightly backups

## Date
2026-10-20

## Time lost
~1h (diagnosis, re-runs, cleanup). One night of backups missed across all four
repos; no data lost, and every repo has a fresh snapshot the same day.

## Status
Resolved. Recurred on the next two nights (2026-10-21 and 2026-10-22, see
Recurrence below), so three nights failed in a row. Prevention proposed below,
not yet implemented.

## Context
- **System / component:** the four restic→B2 backup CronJobs (`nextcloud`,
  `gitea`, `immich`, `paperless`) and the home internet link
- **Scope:** every backup on the cluster for one night. Apps unaffected.
- **State before:** idle overnight. The ISP link dropped twice on 2026-10-20,
  roughly **00:10–08:05 UTC** and **12:00–15:25 UTC**. The schedules are
  03:00 / 03:30 / 04:00 / 04:30 `Europe/Dublin`, i.e. 02:00–03:30 UTC (IST), so
  the whole backup window sat inside the first outage with hours to spare on
  either side.

## Symptoms
Twelve backup alerts on top of the three standing ones (`Watchdog`,
`KubeCPUOvercommit`, `KubeMemoryOvercommit`):

| Alert | Namespaces | Fired (UTC) |
|---|---|---|
| `BackupJobFailed` (critical) | all four | 02:12, 02:32, 03:13, 03:42 |
| `KubeJobFailed` (warning) | all four | 15 min after each of the above |
| `BackupJobMissing` (critical) | all four | 05:07–06:31, i.e. 26h + `for: 1h` after the 2026-10-19 runs |

`BackupJobMissing` is the same missed night crossing its 26h threshold.

None of these could be *delivered* while the link was down. Alertmanager's email
(Brevo SMTP), Pushover and the healthchecks.io deadman all need DNS for an
external name, and all three failed until the link came back, so the first
notification arrived hours after the failures.

## Investigation
- **Cluster health first.** Both nodes `Ready`, no non-`Running` pods outside
  completed Jobs, every app Deployment 1/1. Not a cluster problem.
- **Job logs: gone.** The failed Jobs had no pods
  left to read. All four CronJobs use `restartPolicy: OnFailure`, so retries
  restart the container *inside one pod*. When `backoffLimit` is exceeded, the
  Job controller deletes that pod, and the logs go with it. All that survives is
  `reason: BackoffLimitExceeded` on the Job. Everything below is reconstructed
  from other sources.
- **Where's the repo?** `RESTIC_REPOSITORY` is
  `s3:https://s3.eu-central-003.backblazeb2.com/...`, so every backup needs the
  internet. Combined with the reported outages, that is the obvious hypothesis.
  It needed confirming, because "the internet was down at some point today" does
  not mean it was down from 02:00 to 03:40.
- **Bracketing the outage.** Three independent clocks agreed:
  - **Alertmanager notify failures** are the sharpest. Bucketed per 10 minutes,
    errors start at `00:1x`, run continuously to `08:0x`, stop, then resume
    `12:0x`–`15:2x`. They are all DNS failures through CoreDNS:
    ```text
    dial tcp: lookup smtp-relay.brevo.com on 10.43.0.10:53: server misbehaving
    dial tcp: lookup hc-ping.com on 10.43.0.10:53: server misbehaving
    dial tcp: lookup api.pushover.net on 10.43.0.10:53: server misbehaving
    ```
  - **chrony on `pve`**: `Can't synchronise: no selectable sources (4
    unreachable sources)` at 03:51, a source reselected at 09:16. A lagging
    indicator: chrony takes a while to give up on sources. It brackets the window
    but understates the start.
  - **The ddns log in LXC 101** (only writes on change): no change from
    2026-10-19 02:10 until a new WAN IP at 08:05, and another at 15:30, matching
    the end of each outage. Same tell as in the power-cuts lesson: the router
    re-IPs when the link comes back.
- **Failure times**, from `kube_job_status_failed` in VictoriaMetrics: nextcloud
  02:11, gitea 02:31, immich 03:12, paperless 03:40 UTC. Each Job was dead about
  11 minutes after it started (gitea after 1.5). Every retry budget was spent
  inside an outage that lasted eight hours.
- **Where each script died (inferred from the scripts and the timing, not from
  logs).** The first network call in all four is
  `apk add --no-cache restic` (plus `curl` for gitea) against
  `dl-cdn.alpinelinux.org`, before anything touches B2:
  - **gitea** runs `set -euo pipefail`. `apk add` fails and the script exits
    straight away, *before* the `trap ERR` and the scale-to-0. Gitea was never
    taken down. Two attempts (`backoffLimit: 1`) in ~1.5 min.
  - **nextcloud / immich / paperless** run `set -uo pipefail`, with no `-e` on
    purpose so the retry loops can work. `apk add` fails silently, `pg_dump`
    succeeds (it is in-cluster), and then the restic retry loop retries a binary
    that was never installed: 8 × `restic: not found` + `sleep 20`, so
    ~3 min, then `exit 1`. With `backoffLimit: 3` and the kubelet's restart
    backoff, four attempts come to ~11 min, which matches the observed failure
    times. The retry loops were built for flaky *uploads*. They cannot help when
    the install step has already failed.
- **Is it over?** `curl https://s3.eu-central-003.backblazeb2.com/` from
  `control-node` returned 403 (unauthenticated, which is expected) in 0.09s.
  The link is back.

## Root cause
The internet was down for the whole backup window, and the backups need it
twice: once for `apk add restic`, once for B2. Every Job's retry budget is
measured in minutes (≤ ~12), against an outage measured in hours, so no retry
could have outlasted it. Nothing was misconfigured. This is the expected outcome
of a nightly job that has no catch-up.

## Fix
Re-ran every backup by hand and removed the failed Jobs:

```bash
ssh control-node '
for ns in nextcloud gitea immich paperless; do
  kubectl -n $ns create job --from=cronjob/$ns-backup $ns-backup-manual-$(date +%s)
done
kubectl -n nextcloud delete job nextcloud-backup-29874360
kubectl -n gitea     delete job gitea-backup-29874390
kubectl -n immich    delete job immich-backup-29874420
kubectl -n paperless delete job paperless-backup-29874450'
```

The failed Jobs have to be deleted by hand. `BackupJobMissing` clears on
its own once a backup succeeds, but `BackupJobFailed` and `KubeJobFailed` key on
the failed Job object. `failedJobsHistoryLimit` only evicts it when a *newer*
failure arrives, and the backup CronJobs deliberately carry no
`ttlSecondsAfterFinished`. Left alone, they would have alerted indefinitely (see
`docs/lessons/k8s/kubejobfailed-isolated-failure-never-evicted.md`). No logs were
lost by deleting them: their pods were already gone.

## Verification
```bash
# all four manual Jobs → Complete; each log ends with a 2026-10-20 snapshot:
#   nextcloud-data 24.947 GiB + nextcloud-db   immich-data 26.285 GiB + immich-db
#   gitea-data                                 paperless-media + paperless-db
ssh control-node 'kubectl -n gitea rollout status deploy/gitea'   # scaled back, 1/1
# Alertmanager active alerts → Watchdog, KubeCPUOvercommit, KubeMemoryOvercommit only
```

## Prevention
A missed night is a one-snapshot gap in a 7d/4w/3m retention and every alert
was correct, so most of this is accepted risk. Three cheap improvements, none
applied yet:

1. **`restartPolicy: Never` on the backup Jobs.** Each retry becomes its own
   pod, and failed pods are retained with the Job, so the next failure leaves its
   logs behind. This incident's diagnosis had to be reconstructed from three
   indirect sources because the one direct source had been deleted. Highest
   value, smallest change.
2. **Stop installing restic at runtime.** `apk add` is a second internet
   dependency, and in the no-`-e` scripts its failure is silent and surfaces as a
   misleading `restic: not found` ×8. Either use an image that ships restic, or
   at least fail fast with a clear message when `apk add` fails.
3. **Optionally, wait out short outages.** A pre-flight loop that polls B2 (for
   example every 5 min for up to 2h) before doing any work would have ridden out
   a shorter drop. For gitea it **must** run before the scale-to-0, or a long
   outage turns into Gitea downtime. It would not have saved this night, which
   was 8h, so this is lower value than 1–2.

Re-running is manual and takes about 15 minutes (immich is the long one). It
scales Gitea down for ~1.5 min.

## Related
- Lessons: `docs/lessons/k8s/kubejobfailed-isolated-failure-never-evicted.md` (why the failed Jobs had to be deleted)
- Lessons: `docs/lessons/infra/power-cuts-host-stayed-off.md` (the ddns log as a record of when the link or router dropped)
- Lessons: `docs/lessons/backup/paperless-backup-unmonitored.md` (why all four namespaces alerted, not three)
- Lessons: `docs/lessons/backup/apk-add-hang-no-timeout.md` (the same `apk add` step hanging instead of failing: `DeadlineExceeded` after 4h with the internet up. `restartPolicy: Never` would not have kept that pod's logs)
- ADR: `docs/adr/004-restic-b2-bckup.md`, `docs/adr/019-pushover-critical-alerts.md`
- Runbook: `docs/runbooks/backup-verification.md`

## Recurrence: 2026-10-21 and 2026-10-22

The same failure on each of the next two nights. The alerts stayed up from
2026-10-21 until they were cleared on 2026-10-25. Triage took ~1h.

### Symptoms
Sixteen alerts on top of the baseline three: `BackupJobFailed` and
`KubeJobFailed` once per namespace per night, at the same minutes as on 10-20
(02:12, 02:32, 03:13, 03:42 UTC, each `KubeJobFailed` 14 min later).
`BackupJobMissing` fired for all four from ~21:00 on 10-21, 27h after the 10-20
manual re-runs finished. It cleared between 02:00 and 03:30 on 10-23, as the
scheduled runs succeeded.

### Investigation
- **The backups had already recovered.** All four completed on 10-23, 10-24
  and 10-25, and each latest log ends in `snapshot … saved`. Nothing needed
  re-running. Only the eight failed Job objects were keeping the alerts up, and
  their pods were already gone.
- **Same signature as 10-20.** Every Job hit `BackoffLimitExceeded` the same
  number of seconds after its start as the 10-20 one did:

  | Job | 10-21 failed | 10-22 failed |
  |---|---|---|
  | nextcloud (02:00) | 02:10:34 | 02:10:34 |
  | gitea (02:30) | 02:31:02 | 02:30:57 |
  | immich (03:00) | 03:12:01 | 03:11:51 |
  | paperless (03:30) | 03:40:31 | 03:40:36 |

  From the scripts and the timing, this is the `apk add` failure described
  above. There are no logs to confirm it from.
- **The Alertmanager log had rotated.** It only reached back to 10-22 17:1x
  UTC, so it could not bracket either backup night. The stored metric can,
  because vmsingle keeps 30 days. The Watchdog webhook pings to healthchecks.io
  fail on every attempt while DNS is down, so failed webhook sends are a
  continuous signal:
  ```text
  sum(increase(alertmanager_notifications_failed_total{integration="webhook"}[10m]))
  ```
  Each window below is accurate to ±10 min:

  | Notify failures (UTC) | Host NIC bounce | New WAN IP (ddns) | Hit |
  |---|---|---|---|
  | 10-20 ~19:40 → 19:58 | 19:58 | 20:05 | nothing scheduled |
  | 10-21 ~01:00 → 09:30 | 09:30 | 09:35 | **all four backups** |
  | 10-21 ~16:00 → 16:18 | 16:18 | 16:25 | nothing scheduled |
  | 10-21 ~18:40 → 18:42 | 18:42 | 18:50 | nothing scheduled |
  | 10-21 ~22:50 → 10-22 08:35 | 08:35 | 08:40 | **all four backups** |
  | 10-22 ~14:20 → 20:20 | 20:20 | 20:25 | nothing scheduled |

- **Every outage ended with a router restart.** On `pve`, `journalctl -k` shows
  the same pattern each time: `enp0s31f6: NIC Link is Down` for ~2 min, up for
  ~20 s, down for ~20 s, then up again. A new WAN IP followed 5–7 min later.
  That is the LAN port going away and coming back, which is the router
  restarting. The journal doesn't show whether the router restarted itself or
  someone power-cycled it. Both backup-night outages ended at about 09:30 UTC
  (10:30 local), which fits someone noticing in the morning.
- **Not every re-IP is an outage.** The ddns log also shows new WAN IPs at
  10-24 05:20 and 10-25 04:20 UTC, with no NIC bounce and no failed notifies.
  Those backup nights succeeded.
- chrony's `Can't synchronise: no selectable sources` arrived at 10-21 03:34
  and 10-22 01:24 UTC. It lagged the start of each outage, as on 10-20.

### Fix
Deleted the failed Jobs. No re-run, because a later scheduled run had already
succeeded in every namespace:

```bash
ssh control-node '
kubectl -n nextcloud delete job nextcloud-backup-29875800 nextcloud-backup-29877240
kubectl -n gitea     delete job gitea-backup-29875830 gitea-backup-29877270
kubectl -n immich    delete job immich-backup-29875860 immich-backup-29877300
kubectl -n paperless delete job paperless-backup-29875890 paperless-backup-29877330'
```

### Verification
```bash
# kube-state-metrics stops exporting them immediately:
curl -s 10.42.1.7:8080/metrics | grep -cE '^kube_job_failed\{.*backup'   # 0
# vmsingle keeps returning the last sample for ~5 min, then the alerts resolve.
# amtool alert query → Watchdog, KubeCPUOvercommit, KubeMemoryOvercommit only
```

### Prevention
- **None of the three changes proposed above have been applied, and this is
  now three failed nights in a row.** The case for 1 (`restartPolicy: Never`)
  is stronger: two more nights were diagnosed with no logs at all.
- **A daytime catch-up attempt would have saved both nights.** These outages
  ran from ~01:00 and ~22:50 to 08:35–09:30 UTC, so no retry inside the night
  window could outlast them. The 2h pre-flight wait in item 3 would not have
  either. A second attempt around midday, which runs only if that day's backup
  has not succeeded, would have caught both. Proposed, not applied.
- **Delete the failed Jobs as soon as a later run succeeds.** A successful run
  does not clear `BackupJobFailed` or `KubeJobFailed`. These kept
  re-notifying for four days about backups that had recovered on 10-23.
- The link itself is tracked in `docs/reference/known-risks.md` §11.
