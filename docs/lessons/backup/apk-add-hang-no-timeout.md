# Incident: a stalled `apk add` hung the paperless backup until its 4h deadline

## Date
2026-11-01

## Time lost
~45 min (diagnosis, re-run, write-up). The paperless backup for 2026-11-01
landed 12 hours late, at 16:48 UTC. No data lost.

## Status
Resolved. Prevention proposed below, not yet applied.

## Context
- **System / component:** the `paperless-backup` CronJob (restic → B2), on
  `k3s-worker1`. Schedule `30 4 * * *` `Europe/Dublin`, which is 04:30 UTC
  since the clocks went back on 2026-10-25.
- **Scope:** paperless only. The nextcloud (03:00), gitea (03:30) and immich
  (04:00 UTC) backups that night all succeeded.
- **State before:** idle overnight, internet up.

## Symptoms
Three alerts on top of the baseline (`Watchdog`, `KubeCPUOvercommit`,
`KubeMemoryOvercommit`), all for `paperless`:

| Alert | Fired (UTC) |
|---|---|
| `BackupJobMissing` (critical) | 07:31. The 10-31 run completed at 04:30 UTC, so 26h ran out at 06:31, plus `for: 1h` |
| `BackupJobFailed` (critical) | 08:32, for `paperless-backup-29891790` |
| `KubeJobFailed` (warning) | 08:46 |

The Job had no pod left and therefore no logs. Its status held one clue:

```text
startTime: "2026-11-01T04:30:00Z"
type: Failed   reason: DeadlineExceeded   lastTransitionTime: "2026-11-01T08:30:31Z"
message: Job was active longer than specified deadline
```

A paperless backup normally takes 44–57 s. This one ran for the full
`activeDeadlineSeconds: 14400`.

## Investigation
- **Cluster health.** Both nodes `Ready`, no unhealthy pods. The only ArgoCD
  app off Synced/Healthy was `paperless` (`Degraded`), which a failed CronJob
  run causes by itself.
- **Not the ISP-outage pattern.** The obvious match was
  `internet-outage-backups-failed.md`, and two things ruled it out. First, the
  watchdog-ping failure count was 0 for every 10 min bucket from 04:00 to 09:00
  UTC:
  ```text
  sum(increase(alertmanager_notifications_failed_total{integration="webhook"}[10m]))
  ```
  Second, the signature is different. In an outage the jobs fail fast and hit
  `BackoffLimitExceeded` about 11 min in. This one hit `DeadlineExceeded` after
  4h. The other three backups that night succeeded. In the two logs that
  show the step after the install, restic took 2–3 s to install
  (nextcloud `03:00:03 → 03:00:05`, immich `04:00:03 → 04:00:06`).
- **A hang, not a failure.** `kube_pod_status_phase` shows
  `paperless-backup-29891790-p5nwg` `Running` from 04:31 to 08:30, and
  `kube_pod_container_status_restarts_total` stayed at 0. Under
  `restartPolicy: OnFailure` any `exit 1` restarts the container, so the script
  never exited. For all 4h the container used 0 CPU, its working set sat flat
  at 42 MB, and it received about 4 B/s.
- **Dead end: storage.** `longhorn_volume_robustness` for `paperless-media`
  (`pvc-1aac58d0-…`) was `healthy` and `longhorn_volume_state` `attached`
  throughout. The worker's `journalctl -k` for 04:20–08:40 had no hung-task or
  I/O error lines.
- **Dead end: `pg_dump` waiting on a lock.** Possible in principle, since
  `pg_dump` has no `--lock-wait-timeout`. The postgres pod logged nothing in the
  window, which says nothing either way at default logging. The byte count
  below rules it out: the script never got as far as `pg_dump`.
- **Where it stopped: the byte count.** With no logs, the raw
  `container_network_receive_bytes_total` samples were the best evidence:
  ```text
  04:30:24  1499039     # 20 s after the container started
  04:30:54  1499151
  04:32:54  1499627
  08:29:30  1570873     # 72 KB of background traffic in 4h
  ```
  The previous night's pod had already received about 23 MB (`2.298e+07`)
  by its first sample. A probe pod with the same image
  (`postgres:18-alpine`) measured what `apk add --no-cache` pulls. The main and community `APKINDEX` files come to
  2,980,002 bytes and arrive before any package, and the install with restic
  comes to 17,931,592 bytes in 2 s. 1.5 MB is about half the indexes, so the
  pod stopped partway through the index download, inside
  `apk add --no-cache --quiet restic`. `pg_dump` and restic never ran.
- **Why apk never gave up.** The image is Alpine 3.23.4 with
  `apk-tools 3.0.6`, and `apk --help` there mentions no timeout. A download
  that stops sending without closing the connection leaves apk waiting on a
  read with nothing to end it. The script doesn't wrap the call in a timeout
  either, so the only bound was the Job's 4h deadline. This is inferred from
  the byte count and the idle CPU, not from logs. Why the transfer from
  `dl-cdn.alpinelinux.org` stalled is unknown. Installs before and after it
  worked.
- **The logs went with the deadline.** When `activeDeadlineSeconds` passes,
  the Job controller deletes the running pod. On the worker,
  `/var/log/pods/` held the three previous nights' paperless backup pods and
  not this one.
- **Is it over?** The probe pod installed restic in 2 s at about 16:45 UTC.

## Root cause
At about 04:30:20 UTC, `apk add --no-cache --quiet restic` stalled partway
through fetching the package index from `dl-cdn.alpinelinux.org` and never
returned. Neither apk nor the script puts a timeout on that download, so the
container sat idle until `activeDeadlineSeconds: 14400` failed the Job at
08:30:31. Every backup script installs restic this way, so any of them can
hang like this. The ISP outages fail the same step, but there the jobs give
up within about 11 minutes.

## Fix
No later run had succeeded, so the backup was re-run, and the failed Job was
deleted once its re-run was confirmed. Its pods were already gone, so nothing
was lost by deleting it.

```bash
ssh control-node '
kubectl -n paperless create job --from=cronjob/paperless-backup paperless-backup-manual-$(date +%s)
kubectl -n paperless delete job paperless-backup-29891790'
```

The re-run installed restic in 2 s and saved both snapshots:

```text
44a817b8  2026-11-01 16:47:51  paperless-backup-manual-1793551655-dgwqt  paperless-media  /media  1.595 MiB
c914b320  2026-11-01 16:48:01  paperless-backup-manual-1793551655-dgwqt  paperless-db     /dump   1.348 MiB
```

## Verification
```bash
# the re-run's log: "[backup] Media backup complete", both "snapshot … saved" lines above
# vmalert at 16:51:45 UTC: no alerts beyond the baseline three
# amtool alert query → Watchdog, KubeCPUOvercommit, KubeMemoryOvercommit only
ssh control-node 'kubectl -n argocd get app paperless'    # Synced  Healthy
ssh control-node 'kubectl -n paperless get cronjob paperless-backup -o jsonpath="{.status}"'
# {"lastScheduleTime":"2026-11-01T04:30:00Z","lastSuccessfulTime":"2026-11-01T16:49:59Z"}
```

The ArgoCD `Degraded` cleared on the manual run because
`kubectl create job --from=cronjob/…` gives the Job an owner reference to the
CronJob, so its success counts as the CronJob's `lastSuccessfulTime`.

## Prevention
Proposed, not applied:

1. **Put a timeout on the install** in all four backup scripts:
   ```sh
   timeout 120 apk add --no-cache --quiet restic \
     || { echo "[backup] FATAL: apk add failed or timed out"; exit 1; }
   ```
   `timeout` is in the image (`/usr/bin/timeout`, busybox). A stall becomes
   an `exit 1`, `OnFailure` restarts the container, and the retry opens a new
   connection. That would have saved this night within about 2 minutes. It
   also gives the outage case the fail-fast message that
   `internet-outage-backups-failed.md` item 2 asks for, in place of eight
   `restic: not found` retries. This is the smallest change and it covers
   both failure modes.
2. **Ship restic in the image**, the other half of that item 2. This removes
   the runtime dependency altogether, but someone then has to maintain the
   image. It is the better end state, and item 1 is worth doing first because
   it is cheap.
3. **Give `gitea-backup` an `activeDeadlineSeconds`.** It has none, and it
   runs `concurrencyPolicy: Forbid`. The same stall there would keep the Job
   active indefinitely, and Forbid would then skip every later night until
   someone deleted it. `BackupJobMissing` would be the only signal, 27h in.
   The install runs before the scale-to-0, so Gitea itself would stay up.
   But its `trap … ERR` scale-up runs on an error, not a hang, so a hang
   after `scale 0` would leave Gitea at 0 replicas. This is inferred from the
   manifest. It hasn't happened.

`restartPolicy: Never`, the earlier lesson's item 1, would not have kept this
pod's logs. That item keeps the pods of retries that failed on their own. At
the deadline the Job controller deletes the running pod whatever the restart
policy is. That is from the Kubernetes Job docs, and it matches what happened
here.

## Related
- Lessons: `docs/lessons/backup/internet-outage-backups-failed.md` (the same
  `apk add` step failing fast during an ISP outage. Its prevention items 1 and
  2 are discussed above)
- Lessons: `docs/lessons/k8s/kubejobfailed-isolated-failure-never-evicted.md`
  (why the failed Job had to be deleted)
- Manifests: `k8s/apps/{nextcloud,gitea,immich,paperless}/backup-cronjob.yaml`
- Reference: `docs/reference/operations.md`, Backups (finding where a hung Job
  stopped when it left no logs)
