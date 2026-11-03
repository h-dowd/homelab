# Incident: A GitHub account rename left portfolio and file-parser unpullable, and a reboot found it

## Date
2026-11-03

## Time lost
Portfolio and file-parser were down for ~73 min, from the worker reboot at
21:41 UTC until the fix rolled out at 22:54 UTC. ~20 min of that was
diagnosis, and most of the rest was waiting on me to propose the fix and
on ArgoCD's poll.

## Status
Resolved. The file-parser repo's CI still names the old owner (see Prevention).

## Context
- **System / component:** the `portfolio` and `file-parser` Deployments,
  both self-built images on GHCR (ADR 009).
- **Scope:** those two apps. Portfolio is the public site on the apex.
- **State before:** the first real run of `ansible/upgrade.yml`, rebooting
  both VMs onto new kernels with `--limit k3s_nodes -e reboot=true`. The
  playbook finished with `failed=0`.

## Symptoms
- Both pods stuck in `ImagePullBackOff` after the worker came back, every
  other workload healthy:
  ```text
  failed to resolve reference "ghcr.io/hpdowd/portfolio:a8da99958dd9": failed to authorize:
  failed to fetch anonymous token: unexpected status from GET request to
  https://ghcr.io/token?scope=repository%3Ahpdowd%2Fportfolio%3Apull&service=ghcr.io: 403 Forbidden
  ```
- The playbook's own checks (both nodes Ready, 12/12 volumes
  `attached/healthy`) had all passed. Nothing in its output pointed at the
  two apps.
- Their warnings were pending in vmalert, held back by the 30-minute
  `NodeRecentlyBooted` inhibition.

## Investigation
- **The containerd bind mount didn't come back, so the store was empty.**
  Ruled out. `findmnt /var/lib/rancher/k3s/agent/containerd` showed
  `/dev/vdb[/k3s-containerd]`, and `k3s crictl images` still listed both
  tags. The images were on the node.
- **Why pull at all, then.** Both Deployments set `imagePullPolicy: Always`,
  so the kubelet asks the registry for the tag's digest even when the image
  is cached. The pods that ran before the reboot had pulled long ago and
  never asked again.
- **The package went private.** ADR 009 says the package must stay public.
  But GitGuardian's link earlier the same day was to `github.com/h-dowd/…`,
  while the images were under `hpdowd`. Confirmed a rename:
  ```text
  api.github.com/users/hpdowd    404
  api.github.com/users/h-dowd    200
  ghcr.io/token  hpdowd/portfolio   403    h-dowd/portfolio   200
  ghcr.io/token  hpdowd/file-parser 403    h-dowd/file-parser 200
  ```
  Both deployed tags resolve anonymously under the new owner
  (`ghcr.io/v2/h-dowd/<image>/manifests/<tag>` → 200).
- **ArgoCD looked stuck after the push.** It wasn't. I had the push time
  wrong. It picked up the commit within its normal ~3 min poll.

## Root cause
The GitHub account was renamed from `hpdowd` to `h-dowd`. GitHub redirects
repository URLs and git operations after a rename. GHCR doesn't: the
packages move to the new owner, and `ghcr.io/hpdowd/*` stops resolving,
answering anonymous token requests with 403. Nothing that was already
running noticed. The break waited for the first pod to be created after the
rename. With `imagePullPolicy: Always` that meant any pod creation, cached
image or not, and the worker reboot recreated both. When the rename
happened isn't recorded here.

Portfolio's CI builds the image name from `github.repository_owner`, so
after the rename it pushed to `h-dowd` and its pin step
(`sed "s|(image: ${IMAGE}:).*|…|"`) matched nothing in the manifest. Inferred
from the workflow, not from a CI run: deploys would have stopped silently,
logging `image tag unchanged; nothing to commit`.

## Fix
`89cb1fe`, the owner in both image references:
```diff
-          image: ghcr.io/hpdowd/portfolio:a8da99958dd9
+          image: ghcr.io/h-dowd/portfolio:a8da99958dd9
-          image: ghcr.io/hpdowd/file-parser:sha-a60408b
+          image: ghcr.io/h-dowd/file-parser:sha-a60408b
```
`services.md` and `resources.md` updated to match. ADR 009 is left as
written, since it records the decision, not the current owner.

## Verification
```bash
# ArgoCD on 89cb1fe, both rollouts complete at 22:54 UTC
kubectl -n argocd get app portfolio file-parser   # Synced, 89cb1fe8…
kubectl -n portfolio rollout status deploy/portfolio      # successfully rolled out
kubectl -n file-parser rollout status deploy/file-parser  # successfully rolled out
# the public site through a Cloudflare edge, not the LAN
curl --resolve henrydowd.dev:443:$(dig @1.1.1.1 +short henrydowd.dev | head -1) https://henrydowd.dev/
#   200, 20309 bytes, <title>Henry Dowd — Infrastructure Engineer</title>
# file-parser from inside the cluster
curl http://<file-parser ClusterIP>:<port>/healthz   # {"status":"ok"}
```

## Prevention
- **`upgrade.yml` now waits for every Deployment and StatefulSet to be
  ready** after a reboot, and prints the ones that aren't. This run would
  have failed naming both apps, instead of finishing green.
- **The file-parser repo's CI still pushes to `ghcr.io/hpdowd/file-parser`**
  (`images:` in `.github/workflows/build.yml`), which the new owner can't
  write. Its next build fails until that line says `h-dowd`, or uses
  `${{ github.repository_owner }}` as portfolio's does. That repo is outside
  this one, so it's open.
- **`imagePullPolicy: IfNotPresent` would have kept both apps up.** Both
  tags are immutable SHAs, so `Always` buys nothing but a registry call on
  every pod start. Not applied: it would also have hidden the rename until
  the image was garbage-collected. Proposed, Henry's call.
- **Renaming a GitHub account breaks every `ghcr.io/<old>` reference.**
  `grep -rn 'ghcr.io/' k8s/` lists them.

## Related
- `docs/adr/009-portfolio.md`: "The GHCR package must stay public (or the
  Deployment needs an `imagePullSecret`)."
- `docs/lessons/k8s/act-runner-no-docker-daemon.md`: why the image is built
  on GitHub at all
- `ansible/roles/package_upgrade/tasks/wait_cluster.yml`
