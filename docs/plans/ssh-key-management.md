# SSH authorized keys from the repo

*Drafted 2026-11-17. Not executed. No decision yet: once one is picked, it
becomes ADR 022 and this line says so.*

Every key on every machine went on by hand: p1 onto the worker through
`qm guest exec` on 2026-10-20, the `claude@pve` key onto all three hosts on
2026-11-17, and p1 onto the `claude` user on pve the same day. Nothing
records which key is where, and the audit below found keys nobody can name.
The goal is one list in the repo, and one command that makes every machine
match it, to run whenever a key is added, rotated or revoked.

## What's on each machine (2026-11-17)

Fingerprints from `ssh <alias> 'cat /root/.ssh/authorized_keys' | ssh-keygen -lf -`.

| Key | Fingerprint | pve root | control root | worker root | pve `claude` |
|---|---|---|---|---|---|
| p1, `personal-key (1password)` | `SHA256:RCEAdd1f…` | yes | yes | yes | yes |
| `claude@pve`, restricted `from="192.168.1.2"` | `SHA256:FviJ3fVN…` | yes | yes | yes | — |
| `root@pve`, RSA 4096 | `SHA256:RHz7Dp3F…` | yes | — | — | — |
| `hdowd@crusoe.ai - MBP-Henry-Dowd` | three **different** keys: `nF7YeHBJ…` / `aXwgnzOL…` / `n21Ewod/…` | yes | yes | yes | — |
| `henry@dowd.ie`, no 1Password tag | `SHA256:s4xcx3aB…` | yes | — | — | — |
| `homelab` | `SHA256:uviXMj/G…` | — | yes | yes | — |

- **The MBP key differs on each host.** It has the same comment each time.
  At most one of the three can be the laptop's current key, unless the laptop
  holds three.
- **`s4xcx3aB…` and `homelab` have no known owner.** `homelab` is probably
  the key the VMs were built with. Unverified.
- **`root@pve` is Proxmox's own** and stays. pve's
  `/root/.ssh/authorized_keys` is a symlink to `/etc/pve/priv/authorized_keys`
  on pmxcfs, which Proxmox writes to as well.
- **The LXCs (100, 101, 102) have no `authorized_keys` at all**, though all
  three have `sshd`. They're reached with `pct` from pve.
- pve's file has a blank first line. It's harmless, but it shows the file
  has been edited by hand.

## Options

### A. An Ansible role and a playbook of its own (recommended)

The list of keys goes in `ansible/group_vars/all.yml` as public keys. They
aren't secret, so there's nothing to seal. A `ssh_keys` role writes them,
and `ansible/keys.yml` runs only that role:

```
ansible-playbook -i inventory.ini keys.yml --check --diff
ansible-playbook -i inventory.ini keys.yml
```

That one command covers both "hard-coded" and "a workflow I can run whenever".
It runs from wherever p1 is, the same way `site.yml` and `upgrade.yml` already
do. A new machine gets its keys by joining the inventory. The gateway box
and the Part B box from `docs/plans/gateway-and-capacity.md` will get theirs
for free.

### B. A Gitea Actions workflow (rejected)

`workflow_dispatch` on a "sync keys" job. The runner is a pod in the
cluster (ADR 010), so it would need a private key with root on every machine,
stored as a Gitea secret. That's p1's reach outside 1Password, which the
gateway plan refuses even for the gateway. It also runs on the cluster it's
meant to manage. The time keys most need fixing is when something is broken,
and the cluster may be part of what's broken.

### C. SSH certificates (deferred)

Each machine trusts one CA line (`TrustedUserCAKeys`), and adding a device
means signing its key, not touching a single machine. It's the right shape
at ten-plus machines or with devices that change often. At six machines it
adds a signing step, a CA key to keep in 1Password, and revocation lists to
get right, all to save one playbook run. Revisit after the gateway exists.

### D. Keep doing it by hand, and write the table down

This is the status quo, and it produced the table above.

## Design for A

**No collections.** `ansible.posix.authorized_key` would be the obvious
module, but `ansible.posix` isn't installed here, and `ansible/ansible.cfg`
already records one run broken by a module moving out of ansible-core. The role
uses `ansible.builtin` only:

- **Phase 1, additive:** `lineinfile` per key. It adds and never removes, so
  it's safe to run before the unknown keys are settled.
- **Phase 2, exclusive:** `template` writes the whole file. Anything not in
  the list is gone, which is what makes revoking a key a one-line change.

**The list:**

```yaml
# group_vars/all.yml
ssh_keys:
  p1: "ssh-ed25519 AAAA… personal-key (1password)"
  claude_pve: 'from="192.168.1.2" ssh-ed25519 AAAA… claude@pve'
  mbp: "ssh-ed25519 AAAA… hdowd@crusoe.ai - MBP-Henry-Dowd"

# who gets which, per host or group
ssh_authorized:
  root: [p1, claude_pve, mbp]
```

pve's `host_vars` adds `root@pve` to root's list and a `claude: [p1]`
entry. Key names, not key strings, go in the per-user lists, so rotating a
key is one edit.

**pve is the awkward one:**

- **The symlink.** `template` and `copy` replace a symlink with a regular
  file unless given `follow: true`. Replacing it would detach root's keys
  from the file Proxmox maintains. For pve, the role writes to
  `/etc/pve/priv/authorized_keys` directly.
- **pmxcfs** doesn't behave like a normal filesystem for ownership and
  mode changes. Untested with `template`; `--check --diff` against pve
  comes first.
- **Host configuration.** The host's configuration has been by hand, and
  `upgrade.yml` is the only playbook that targets it (`ansible/README.md`).
  `keys.yml` would be the first to change files on it. That's a deliberate
  exception for one file, and the README says so when this lands.

**The LXCs stay out of v1.** They have no keys and are reached through
`pct` from pve, so there's nothing to manage. If one ever needs direct SSH,
it joins the inventory and gets a list like any other host. Anything run
against 101 follows the `homelab-access` rule about the VPN LXC.

## Steps

1. **Settle the unknown keys.** On the MacBook, `ssh-add -l` and
   `ssh-keygen -lf ~/.ssh/*.pub` find which of the three MBP fingerprints is
   current. Decide `s4xcx3aB…` and `homelab`: keep with a name, or drop.
   Success: every row in the table above has an owner or a "remove".
2. **The role and `keys.yml`, additive.** `--check --diff` against all three
   hosts. Success: the diff only adds lines, and adds none that are already
   present.
3. **Run it.** Success: `ssh -o BatchMode=yes <alias> true` still works for
   all three, from the workstation (p1) and from `claude@pve`.
4. **Switch to exclusive.** Keep a root console open first: the Proxmox web
   UI for pve, `qm terminal 300`/`301` for the VMs. `--check --diff` should
   show only the removals decided in step 1. Run it, then repeat step 3's
   checks.
5. **Write it up.** ADR 022 (A over B, C and D), an `ansible/README.md`
   section, and a line in `homelab-access` replacing the
   `qm guest exec` recipe with `keys.yml`.

## Open questions

- Should `keys.yml` fold into `site.yml`? Keeping it separate means a key
  change never touches k3s config, and `site.yml` never touches pve. I'd
  keep it separate.
- Is the `claude@pve` key's `from=` restriction enough once the gateway
  exists, or does Claude move to the gateway with its own key? That depends
  on the gateway plan's open question about where sessions run.
