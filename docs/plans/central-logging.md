# Central logging with VictoriaLogs

*Drafted 2026-11-17. Not executed. Decision: `docs/adr/021-victorialogs-central-logging.md`.*

It can go in before or after the gateway plan's Part B. The VictoriaLogs
address stays the same when the worker moves.

## Before starting

Checked 2026-11-18. Re-run them if the plan sits for weeks.

- `free -h` on the worker and `kubectl describe node k3s-worker1`.
  VictoriaLogs plus Vector add about 300MiB of limits to a node that is
  already overcommitted (known-risks §4). On 2026-11-18: 3.9Gi available,
  requests 6123Mi (47%), limits 23652Mi (183%), about 185% after this plan.
- `ssh pve 'zfs list tank'`: room for the 8GiB cap. `local-path` writes to
  `/mnt/longhorn/local-path`, which is the worker's `vdb` on ZFS `tank`, not
  `pve/data`, so known-risks §10 doesn't apply to the store. 1.09T free on
  2026-11-18.
- `ls /var/log/journal` on each machine. On 2026-11-18 it existed on pve, both
  VMs and LXCs 100 and 102. LXC 101 is Alpine, with no systemd and no journal
  (§3b).
- LXC 102 (AMP)'s root disk, `vm-102-disk-0`, was 93% full of 61G on
  2026-11-18, with a 514M journal and no `SystemMaxUse`. Free space there, or
  cap the journal, before step 3 relies on it as the buffer.
- 192.168.1.201 is unused in the MetalLB pool (.200–.210). Only Traefik's .200
  was taken on 2026-11-18.

## 1. VictoriaLogs on the cluster

- An ArgoCD Application for the `victoria-logs-single` chart in `monitoring`.
  Image pinned, worker `nodeSelector`, securityContext and limits to the
  baseline (ADR 011, ADR 020).
- `local-path` PVC 10Gi. `retentionPeriod: 30d`,
  `retention.maxDiskSpaceUsageBytes: 8GiB`.
- Service `LoadBalancer`, `metallb.universe.tf/loadBalancerIPs: 192.168.1.201`,
  port 9428/TCP and 514/UDP, `externalTrafficPolicy: Local`. Without `Local`,
  kube-proxy rewrites the source to a node IP and the NetworkPolicy below
  can't tell the hosts apart.
- The UDP port is for LXC 101 (§3b). VictoriaLogs listens with
  `-syslog.listenAddr.udp=:5140` (unprivileged, mapped to 514 by the Service)
  and `-syslog.timezone=UTC`, because busybox's RFC 3164 lines carry no year
  or zone. If the chart's Service can't take the extra port, use a second
  Service on the same IP (`metallb.universe.tf/allow-shared-ip`). Check both
  flag names against the chart before relying on them.
- NetworkPolicy ingress allows only 192.168.1.2, .5, .3, .10, .11 and .15,
  plus the Grafana and vmalert pods.
- The chart's Vector DaemonSet is enabled, with the same worker
  `nodeSelector`.
- Grafana: add the `victoriametrics-logs-datasource` plugin and a datasource
  for it in `k8s/apps/monitoring.yaml`.
- Add a row to `docs/reference/resources.md`.

Done when `curl http://192.168.1.201:9428/health` returns `OK` from pve, and a
worker pod's logs show up in Grafana Explore.

Then record what it really uses: `kubectl top pod -n monitoring` for the
VictoriaLogs and Vector pods, once at rest and once while a 30-day Grafana
query runs. Estimated 2026-11-18, not measured: VictoriaLogs ~80–150Mi, Vector
~40–80Mi, from ~220k lines (~60MB) a day across all hosts and pods. Set the
VictoriaLogs limit to 256Mi; it sizes its caches from the limit
(`-memory.allowedPercent`), so usage under query load tracks the limit.

## 2. Journals from the VMs (Ansible)

A new role, `journal_upload`, added to `site.yml`:

- `apt install systemd-journal-remote`
- `/etc/systemd/journal-upload.conf.d/10-victorialogs.conf`:
  `URL=http://192.168.1.201:9428/insert/journald`
- `Storage=persistent` added to the existing drop-in in `roles/common`, next
  to `SystemMaxUse`
- `systemctl enable --now systemd-journal-upload`

Done when `_HOSTNAME:k3s-control` and `_HOSTNAME:k3s-worker1` both return
lines from the last 5 minutes.

## 3. Journals from pve and the LXCs (by hand)

The host is still configured by hand (`ansible/inventory.ini`). Do the same
steps as in 2 on pve, then on LXCs 100 and 102 with `pct exec`. Record what
was changed in `docs/reference/operations.md`.

Done when each hostname returns recent lines.

### 3b. LXC 101 over syslog

LXC 101 (WireGuard) is Alpine. It has busybox `syslogd` writing to
`/var/log/messages`, and no journal. **Only `pct exec`, never `pct enter`,
while on the VPN** (homelab-access).

- `/etc/conf.d/syslog`: `SYSLOGD_OPTS="-t -L -R 192.168.1.201:514"`. `-L`
  keeps writing `/var/log/messages` as well, and that file is the only copy
  of anything sent while the store is down: UDP syslog has no buffer and no
  retry.
- `rc-service syslog restart`
- `cloudflare-ddns.sh` appends straight to `/var/log/cloudflare-ddns.log` and
  never reaches syslog. Add `logger -t cloudflare-ddns "<message>"` next to
  each write to that file. The script isn't in the repo yet. Copy it into
  `lxc/wireguard/` first, next to `vpn-healthcheck.sh`.

Done when `hostname:wireguard` (or whatever host field VictoriaLogs' syslog
parser sets: check one line) returns a recent `cloudflare-ddns` line.

## 4. One alert

A rule that alerts when a host has sent nothing for 30 minutes,
`severity: warning`. VictoriaLogs' own metrics aren't labelled by host, so
this is probably a vmalert `vlogs` rule counting lines by `_HOSTNAME`, with
LXC 101 matched on the syslog host field instead. 101 logs little, but
`crond` writes a `cron.info` line for every 5-minute ddns and healthcheck run
(checked 2026-11-18), so 30 minutes of silence still means it stopped
sending. Work out the exact query when building it. To test it, stop
`systemd-journal-upload` on one VM and check that the alert fires. Log-content
alerts (`Hardware Error`, `Detected Hardware Unit Hang`) wait until the
pipeline has run for a few weeks.

## 5. Test the outage path

Stop the VictoriaLogs pod for 10 minutes and run `logger central-logging-test`
on pve. Once the pod is back, the line arrives with its original timestamp.
A line sent from LXC 101 in that window never reaches the store and survives
only in its `/var/log/messages`. That is expected (§3b).

## On completion

- Mark this plan executed and add `## Verification` to ADR 021, with the
  measured memory from step 1 and the store's disk use after a week.
- Add the query commands to `docs/reference/operations.md` (curl
  `/select/logsql/query` from the LAN), so sessions use them instead of SSH.
