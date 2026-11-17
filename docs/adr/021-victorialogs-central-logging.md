# ADR 021: Logs from every machine go to VictoriaLogs on the cluster, and stay on disk where they were written

**Status:** Accepted
**Date:** 2026-11-17

## Problem

- Each machine keeps its logs on its own disk: pve, both k3s VMs, and the
  three LXCs. Reading them takes a root shell on each one. Most of the
  incidents in `docs/lessons/infra/` were diagnosed from host logs.
- Claude's sessions run as an unprivileged `claude` user on pve. Without
  central logs, diagnosing a host problem needs root on the hypervisor.
  Root should only be needed for fixes, which Henry grants for each task.
- Pod logs go when the pod goes. `kubectl logs --previous` covers one restart,
  not a deleted pod.

## Decision

- **Store: VictoriaLogs single-node** in the `monitoring` namespace, pinned
  to the worker. It goes on `local-path`, which is on `vdb`
  (known-risks §3), for the same reason as the metrics TSDB (ADR 005
  decision 2): the data is disposable. Retention is 30 days with a disk cap.
- **Hosts and VMs ship their journal with `systemd-journal-upload`**, which
  posts to VictoriaLogs' journald endpoint. It records how far it has sent
  and resumes from there, so the local persistent journal is the buffer. A
  cluster outage delays logs, and doesn't lose them, as long as the journal
  still holds them.
- **Pod logs:** one Vector DaemonSet, **worker only** (ADR 020). The control
  node runs only k3s system pods, and k3s itself logs to the journal, which is
  already shipped.
- **Every machine keeps a persistent local journal.** Logs from just before a
  crash may never be shipped. `journalctl -b -1` on the machine itself is
  what catches them.
- **Reached on the LAN** at a MetalLB address. A NetworkPolicy admits only the
  homelab's own hosts, Grafana and vmalert.

## Rejected

- **Loki (with Promtail or Alloy):** a second query language next to MetricsQL,
  and its single-binary mode wants object storage to be comfortable. Promtail
  is deprecated.
- **OpenSearch or ELK:** needs gigabytes of heap. Host RAM is full
  (`docs/plans/gateway-and-capacity.md`).
- **rsyslog to a file on one VM:** central, but it can't be searched across
  hosts or alerted on.
- **The store as an LXC on pve:** no RAM for it on pve. It would still fail
  with pve, and it would sit outside GitOps.
- **The store on the gateway box (that plan's Part A):** the gateway is kept
  minimal so it survives what kills everything else. A database writing to its
  SSD works against that.
- **Vector on every host instead of `systemd-journal-upload`:** another agent
  to install and pin on each host. Journal-upload is a Debian package, and the
  journal is all the hosts produce.

## Consequences

- **A pve failure takes down the store along with the machine whose logs
  matter most.** Upload resumes after the reboot. Lines never written to disk
  are lost, so the local journal is the only record of a crash. This
  mostly goes away once the worker moves to the new box
  (`docs/plans/gateway-and-capacity.md` Part B).
- **VictoriaLogs has no auth.** Anything on the hosts allowed by the
  NetworkPolicy can read every log, and logs can hold secrets. The admitted
  sources are the same machines that already hold root keys.
- **A larger `pve/data` thin pool.** The disk cap bounds the growth;
  known-risks §10 still applies.
- **Claude can diagnose pve from its logs without a shell on pve.**
