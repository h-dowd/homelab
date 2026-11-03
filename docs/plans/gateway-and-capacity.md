# A gateway box for access, and more capacity for the cluster and AMP

*Drafted 2026-11-03. Not executed. Hardware shopping 2026-11-04.*

Two pieces of work that share a shopping trip. Part A is a gateway box for
access. Part B moves the k3s VMs to an older box with more RAM, so AMP keeps
the host's fast CPU.

## Shopping list

| Item | Why | Notes |
|---|---|---|
| Raspberry Pi 5, 8GB, or any small x86 box | the gateway (Part A) | arm64 is fine: every tool it runs has an arm64 build. Boot from NVMe or USB SSD, not an SD card, which is the part that dies |
| Official PSU and a case with a fan | the gateway | under-voltage on a Pi shows up as flaky USB storage, which is the gateway's disk |
| A smart plug with **local** control, no cloud needed | power-cycling the host from the gateway | e.g. Shelly Plug S, or anything Tasmota runs on. It must be switchable over the LAN with the internet down, since that's when it's needed |
| A small UPS | router and gateway | covers short power cuts. The host has AC Recovery on (2026-10-08) and comes back by itself |
| An older x86 box with lots of RAM, DDR3 is fine | the k3s VMs move onto it, AMP keeps the host (Part B) | **check `grep -o avx2 /proc/cpuinfo` on it before buying** (Part B). 1GbE. An SSD for the Longhorn disk. Runs Proxmox. Not the gateway |

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

### The split

More RAM in the host is out: it takes DDR5 SODIMMs only, and they cost too
much for this (Henry, 2026-11-03). The capacity comes from an older box with
more, cheaper RAM, and the two boxes split by what suits each:

| Box | Runs | Why |
|---|---|---|
| The host (i5-14500T) | AMP, which grows into the freed RAM. Technitium | a game server is single-thread bound, and nothing in the cluster uses this CPU's speed |
| The new box (older, more RAM) | VM 301, the worker. Later VM 300, control | the cluster wants RAM, not clock speed |

Moving VM 301 frees 13GiB on the host for AMP. Moving VM 300 too frees
another 4GiB, and then a host reboot for AMP no longer takes the cluster
down.

### Move the VMs whole, not redesign the cluster

Back VM 301 up with `vzdump` and restore it on the new box's Proxmox, with
the same VMID, MAC and IP. Inside the VM nothing changes: hostname
`k3s-worker1`, `192.168.1.11`, the ADR 020 pins and the Longhorn disk on
`vdb` all carry over. No ADR is needed, because the design doesn't change,
only the hardware under it.

- **The copy.** `vdb` holds 142GB of Longhorn data (`df /mnt/longhorn`,
  2026-11-03), plus the 44GiB OS disk. Over the host's current 100 Mb/s link
  (known-risks §11, open action 19) that's about 4–5 hours of worker
  downtime. At 1 Gb/s it's about 25 minutes. **Fix the link first**, or
  shuttle the backup on a USB disk.
- **Shut down through the playbook's path.** Stop the worker with k3s
  running, so the graceful shutdown applies (gotchas.md, "k3s node
  operations"), before `vzdump` with `--mode stop`.
- **Set the startup options by hand on the new host**: `onboot: 1`,
  `startup: order=…,up=30,down=300`, `agent: 1`. The 300s matters, see
  `k3s_shutdown_grace_by_priority` in `ansible/group_vars/all.yml`.
- **The storage name.** `vdb` is on the host's `tank` pool. Name the new
  box's storage `tank` too, or map it at restore time.
- Delete the old copy on the host only after the restored worker has run
  for a week.

### Check the CPU before buying

The worker runs with `cpu: host`, so it sees the i5-14500T's AVX2.
Intel CPUs before Haswell (4th gen, 2013) don't have it, and that covers a
lot of DDR3 hardware, including the Xeon E5 v1/v2 workstations that take the
most DDR3. Nothing in the repo records which images need it. Before buying
anything pre-Haswell, check the Immich requirements (the ML image and the
vectorchord postgres), the heaviest images here. Haswell desktops and Xeon
E3 v3 have AVX2 and still use DDR3, but top out at 32GB.

Old hardware also costs power: an old tower idles at several times what the
OptiPlex Micro does.

### What changes in the repo

- `ansible/upgrade.yml` assumes both VMs live on the one host: the host
  reboot covers their kernels, and its guest checks run `qm config` there. It
  needs a second Proxmox host in the inventory, and each VM to know which
  host it's on.
- `ProxmoxHostRestarted` reads "both VMs booted within an hour" as a host
  restart. With the VMs on different hosts, that inference is wrong and the
  rule needs rethinking.
- `docs/reference/architecture.md`, `ansible/inventory.ini`'s VM sizes, and
  the homelab-access guest table.

### Rejected

- **More DDR5 in the host.** Cost.
- **The new box as a second worker** (two workers, replicas across boxes).
  It buys redundancy as well as capacity, but the design changes with it:
  ADR 020's hostname pins become a label, Longhorn goes from one replica to
  two with sync over the network, and the control plane is still one VM.
  Worth its own ADR if redundancy is ever the goal. Moving the VMs whole
  doesn't rule it out later.
- **AMP on the new box.** It would get the slower CPU, which is the one
  thing a game server cares about.
- **A two-node Proxmox cluster.** It loses quorum when either node is down,
  and then neither starts guests. Two standalone hosts, each with its own
  guests.

## On completion

Mark this executed at the top, with links to the gateway ADR and to any
lessons the VM move produced.
