# A gateway box for access, and more capacity for the cluster and AMP

*Drafted 2026-11-03. Not executed. Hardware shopping 2026-11-04.*

Two pieces of work that share a shopping trip. Part A, the gateway, is the
one this plan scopes in full. Part B, capacity, settles what to buy, and
leaves a second worker to its own ADR if it is ever wanted.

## Shopping list

| Item | Why | Notes |
|---|---|---|
| Raspberry Pi 5, 8GB, or any small x86 box | the gateway (Part A) | arm64 is fine: every tool it runs has an arm64 build. Boot from NVMe or USB SSD, not an SD card, which is the part that dies |
| Official PSU and a case with a fan | the gateway | under-voltage on a Pi shows up as flaky USB storage, which is the gateway's disk |
| A smart plug with **local** control, no cloud needed | power-cycling the host from the gateway | e.g. Shelly Plug S, or anything Tasmota runs on. It must be switchable over the LAN with the internet down, since that's when it's needed |
| A small UPS | router and gateway | covers short power cuts. The host has AC Recovery on (2026-10-08) and comes back by itself |
| A used x86 mini PC, DDR4, 32GB | AMP now, maybe a worker VM later (Part B) | the 8th–10th gen business minis (OptiPlex/EliteDesk/ThinkCentre Micro) take DDR4 SODIMMs, which cost far less than the host's DDR5. Run Proxmox on it. Not the gateway: see Part B |

## Part A: the gateway

### Why

- **Remote access runs on the host it's meant to rescue.** WireGuard is LXC
  101 on the Proxmox host. On 2026-11-03 the upgrade run couldn't reboot the
  host, because the workstation was off the LAN and the tunnel goes down with
  the host. `ansible/upgrade.yml` now refuses to try. On 2026-09-03 LXC 101
  didn't come back after a reboot (known-risks §9). The `pct enter` D-state
  freeze (`docs/lessons/infra/wireguard-lxc-dstate-freeze.md`) only exists
  because the VPN is a container on the host being managed.
- **The tools live wherever the session happens to be.** On 2026-11-03 the
  workstation had no Ansible, and `kubeseal` was installed nowhere, while
  operations.md's sealing procedure assumed it. Each new workstation
  rediscovers this.
- **Nothing can power-cycle a hung host remotely.** The e1000e NIC hang and
  the D-state freeze both end in a hard reboot by hand.

### What runs on it

| Runs on the gateway | Why there |
|---|---|
| Tailscale, as a subnet router for 192.168.1.0/24 | remote access that survives the host going down |
| The maintenance tools: `ssh`, `git`, `ansible-core`, `python3`, `curl`, `dig`, Claude Code | one machine that always has them. `kubectl` and `kubeseal` stay on `control-node` |
| A clone of this repo | Ansible runs from here |
| Smart-plug control for the host | the remote hard reboot |
| An external watcher: ping the host and both VMs, report to healthchecks.io | the out-of-band check that known-risks §9 and the e1000e lesson ask for. It can see the host from outside the host |

**Not on it: cloudflared.** It carries every public request into the
cluster. On the gateway, the box holding the admin keys would be on the
public data path, for no gain: when the cluster is down the sites are down
anyway, because their backends are in the cluster. cloudflared stays in
`k8s/infrastructure/cloudflared/`, under GitOps.

### Keys

- **The gateway gets its own key**, added to `authorized_keys` on `pve`,
  `control-node` and `worker-node` the way homelab-access describes. A lost
  or compromised gateway is revoked by removing one line on three machines.
- **p1 stays in 1Password.** It is never copied to the gateway. The
  workstation uses it to log in to the gateway.
- **`ProxyJump`, not agent forwarding.** The workstation's `~/.ssh/config`
  sends `pve`, `control-node` and `worker-node` through the gateway, so
  `ssh pve` works unchanged from anywhere. With agent forwarding, root on the
  gateway could use p1 while you're connected.
- **Commits made on the gateway** need a signing key. Open question: a
  gateway signing key registered on Gitea, or commit only from a
  workstation.

### Tailscale or WireGuard

Tailscale, with WireGuard kept as the fallback during cutover.

- **No port forward.** The Vodafone hub has silently wiped one before
  (`docs/lessons/networking/vodafone-hub-ghost-portforward.md`).
- **No DDNS.** `home.henrydowd.dev` and the cloudflare-ddns job in LXC 101
  can be retired with it.
- **Works behind CGNAT**, if the ISP ever moves there.
- **The desktop already runs it.**
- **The cost** is depending on Tailscale's coordination service. Headscale
  would remove that, but needs hosting somewhere that isn't the homelab.

Disable key expiry for the gateway's node, or it drops off the tailnet after
180 days. Name resolution: Technitium (LXC 100) is on the host, so use the
ssh aliases and IPs for admin work, not names that depend on it.

### Cutover

Each step is reversible until step 7. LXC 101 keeps running alongside until
then.

1. **Build the gateway from Ansible**, a new `gateway` group and role, so it
   can be rebuilt from the repo. Packages, the tools above, the repo clone,
   the external watcher. Success: `ansible-playbook site.yml --check --diff`
   reports `changed=0` on it after the first run.
2. **Give it a key.** Generate on the gateway, append to `authorized_keys` on
   the three machines (via `qm guest exec` for the VMs, as homelab-access
   describes). Success: `ssh pve true` from the gateway.
3. **Tailscale.** Join, advertise 192.168.1.0/24, approve the route,
   disable key expiry. Success: from a phone on mobile data, with WireGuard
   off, `ssh` to the gateway and from it to `pve`.
4. **ProxyJump** in the workstation's ssh config. Success: `ssh pve`,
   `ssh control-node` and the Ansible playbooks work from off the LAN with
   WireGuard off.
5. **The smart plug.** Pair it on the LAN, script on/off from the gateway.
   Test with the host shut down cleanly first, not by cutting power under
   load. Success: power off and on from the gateway, and the host boots.
6. **The watcher.** A check from the gateway every few minutes, pinging a
   healthchecks.io URL. Success: stop the check and the alert arrives.
7. **Retire LXC 101.** Needs Henry: it touches the router (remove the port
   forward), Cloudflare (`home.henrydowd.dev`) and LXC 101 itself. Stop it,
   leave it for a week with `onboot: 0`, then delete it.

### Repo changes, once cut over

- `CLAUDE.md`: rule 2 (`pct enter` into 101) retired with 101. Rule 3 and the
  workstation section of `homelab-access` point at the gateway.
- `ansible/upgrade.yml`: the reboot-over-VPN guard keys on the route going
  through `homelab`. It should check instead that the session doesn't depend
  on the host, or go once 101 is gone.
- `docs/reference/known-risks.md` §9, and the WireGuard section of
  `gotchas.md`: rewritten, with dated updates.
- `docs/reference/architecture.md`: the gateway as a new box outside the
  host.
- An ADR for the gateway, with the rejected options below.

### Rejected

- **The gateway as a VM or LXC on the host.** Same failure domain as now.
- **cloudflared on the gateway.** Covered above.
- **Copying p1 to the gateway, or forwarding the agent.** Covered above.
- **WireGuard on the router.** The Vodafone hub doesn't offer it, and it has
  lost config before.

## Part B: capacity

### Where the RAM is

Read 2026-11-03 from `pve`. The host is a Dell OptiPlex Micro 7020, i5-14500T
(20 threads), 24GB: one 8GB and one 16GB DDR5 SODIMM. The pair is
mismatched and runs at 4800 MT/s. The board takes 64GB in two slots.

| Guest | Allocated | Notes |
|---|---|---|
| VM 301 k3s-worker1 | 13GiB | every workload |
| VM 300 k3s-control | 4GiB | control plane only (ADR 020) |
| LXC 102 AMP | 5GiB | ~2GiB used, 3GiB available inside it |
| LXC 100 Technitium | 1GiB | |
| LXC 101 WireGuard | 256MiB | goes with Part A |
| ZFS ARC | up to 2.3GiB | `zfs_arc_max` |

`free` on the host: 19GB used, 3GB available. Every guest's allocation
already adds up to more than the host has left.

### Options

More RAM in the host is out: it takes DDR5 SODIMMs only, and they cost too
much for this (Henry, 2026-11-03). So the capacity comes from a second box.

1. **AMP moves to the second box, outside the cluster.** The box runs
   Proxmox, and LXC 102 is restored onto it from a backup. That frees 5GiB
   on the host, which the worker can take (13GiB → 18GiB), and AMP gets the
   new box's RAM. Nothing in the cluster changes. This is the
   recommendation.
2. **The second box as a second k3s worker.** This buys redundancy as well
   as capacity, but only if the design changes with it. It needs its own
   ADR. Before choosing it, know that it means:
   - **ADR 020 pins every workload to `k3s-worker1` by hostname.** A new
     worker gets nothing until the pins become a label.
   - **Longhorn runs one replica per volume** (`defaultReplicaCount: 1` in
     `k8s/infrastructure/longhorn.yaml`). Redundancy needs two, across
     boxes, and replica sync crosses the network.
   - **The host's link is at 100 Mb/s** (known-risks §11, open action 19).
     Fix that first, or replica rebuilds crawl.
   - **It must be x86_64.** The self-built images are built for amd64.
   - **The control plane stays on the host**, so the host is still the
     single point of failure for the cluster.

Buying a box that runs Proxmox keeps both open: AMP goes on it now, and a
worker VM can join it later if option 2 is ever wanted, after its ADR.

**Two standalone Proxmox hosts, not a Proxmox cluster.** A two-node Proxmox
cluster loses quorum when either node is down, and then neither can start
guests. Fixing that needs a third vote (a QDevice), which is more moving parts
than this needs.

**The AMP box is not the gateway.** AMP is a game server with ports open to
players. The gateway holds the admin keys and should expose nothing but
Tailscale. Keep them on separate hardware.

**Moving AMP touches:**
- the `amp-external` EndpointSlice, which points Traefik at 192.168.1.15
  (`docs/reference/architecture.md`). If the LXC keeps its IP on the new box,
  nothing changes. If not, the slice and its Authelia rule move with it.
- the router's port forwards for the game ports, if they target 192.168.1.15.
- open action 17 (`fsck` `vm-102-disk-0`), which is easiest done during the
  move, when the container is stopped anyway.

## On completion

Mark this executed at the top, with links to the gateway ADR, and to the
second-worker ADR if option 2 ever goes ahead.
