# Central logging with VictoriaLogs

*Drafted 2026-11-17. Not executed. Decision: `docs/adr/021-victorialogs-central-logging.md`.*

It can go in before or after the gateway plan's Part B. The VictoriaLogs
address stays the same when the worker moves.

## Before starting

- `free -h` on the worker and `kubectl describe node k3s-worker1`.
  VictoriaLogs plus Vector add about 300MiB of limits to a node that is
  already overcommitted (known-risks §4).
- `ssh pve 'lvs pve/data'`: the thin pool has room for the disk cap (§10).
- `ls /var/log/journal` on each machine. pve has it (checked 2026-11-17). The
  VMs and LXCs are unchecked.

## 1. VictoriaLogs on the cluster

- An ArgoCD Application for the `victoria-logs-single` chart in `monitoring`.
  Image pinned, worker `nodeSelector`, securityContext and limits to the
  baseline (ADR 011, ADR 020).
- `local-path` PVC 10Gi. `retentionPeriod: 30d`,
  `retention.maxDiskSpaceUsageBytes: 8GiB`.
- Service `LoadBalancer`, `metallb.universe.tf/loadBalancerIPs: 192.168.1.201`,
  port 9428, `externalTrafficPolicy: Local`. Without `Local`, kube-proxy
  rewrites the source to a node IP and the NetworkPolicy below can't tell the
  hosts apart.
- NetworkPolicy ingress allows only 192.168.1.2, .5, .3, .10, .11 and .15,
  plus the Grafana and vmalert pods.
- The chart's Vector DaemonSet is enabled, with the same worker
  `nodeSelector`.
- Grafana: add the `victoriametrics-logs-datasource` plugin and a datasource
  for it in `k8s/apps/monitoring.yaml`.
- Add a row to `docs/reference/resources.md`.

Done when `curl http://192.168.1.201:9428/health` returns `OK` from pve, and a
worker pod's logs show up in Grafana Explore.

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
steps as in 2 on pve, then on LXCs 100, 101 and 102 with `pct exec`. **Never
`pct enter` 101 while on the VPN** (homelab-access). Record what was changed in
`docs/reference/operations.md`.

Done when each hostname returns recent lines.

## 4. One alert

A rule that alerts when a host has sent nothing for 30 minutes,
`severity: warning`. VictoriaLogs' own metrics aren't labelled by host, so
this is probably a vmalert `vlogs` rule counting lines by `_HOSTNAME`. Work
out the exact query when building it. To test it, stop
`systemd-journal-upload` on one VM and check that the alert fires. Log-content
alerts (`Hardware Error`, `Detected Hardware Unit Hang`) wait until the
pipeline has run for a few weeks.

## 5. Test the outage path

Stop the VictoriaLogs pod for 10 minutes and run `logger central-logging-test`
on pve. Once the pod is back, the line arrives with its original timestamp.

## On completion

- Mark this plan executed and add `## Verification` to ADR 021.
- Add the query commands to `docs/reference/operations.md` (curl
  `/select/logsql/query` from the LAN), so sessions use them instead of SSH.
