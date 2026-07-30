## Navidrome
Self-hosted music streaming server (Subsonic-compatible), deployed to the `pi-cluster` via Flux/Kustomize and exposed through the shared Cloudflare Tunnel.

### Architecture
- **Base manifests** (`apps/base/navidrome/`): Namespace, PersistentVolumeClaim (app data only), Deployment, and Service — environment-agnostic.
- **Staging overlay** (`apps/staging/navidrome/`): applies the `navidrome` namespace via Kustomize. No Secret needed — unlike Vaultwarden, Navidrome requires no sensitive env vars to run.
- **Storage — two different mechanisms, deliberately**:
  - App data (SQLite db, cache, generated artwork) uses a PVC mounted at `/data`, same pattern as Vaultwarden's own data volume.
  - The actual music library uses a `hostPath` volume mounted **read-only** at `/music`, pointing directly at `/mnt/music/Music` on the `gunnlab` node — not a PVC, since this is pre-existing data on a specific physical drive, not something Kubernetes should provision or manage.
- **Ingress**: one more `ingress` rule added to the same shared `cloudflared` ConfigMap used by Linkding and Vaultwarden, routed to Navidrome's Service via its fully-qualified cross-namespace address.
- **Reconciliation**: picked up automatically by Flux's existing `apps/staging/` watch — no new Flux Kustomization CR required.

### Notable issues hit during setup (and how they were diagnosed)

**1. External library lives on exFAT, which has no Unix ownership model**
The music library sits on an external SSD formatted exFAT (originally used on macOS), which stores no per-file Unix owner/permissions — everything is governed entirely by the mount's `uid`/`gid`/`umask` options. Fixed by adding a permanent `/etc/fstab` entry (`uid=1000,gid=1000,umask=022,nofail`) instead of relying on the desktop session's ephemeral `udisks2` auto-mount, ensuring the mount is both stable across reboots and readable by the container running as UID 1000.

**2. Duplicate `volumeMounts` key in the Deployment**
An early draft of the Deployment defined `volumeMounts:` twice under the same container instead of one list with two entries — valid-looking YAML that would have silently dropped the music mount at parse time. Caught by review before applying, not by a runtime failure.

**3. Leftover `envFrom` reference to a nonexistent Secret**
Copying Vaultwarden's Deployment as a starting point carried over its `envFrom: secretRef: vaultwarden-container-env`-style block, pointed at a Navidrome secret that was never created (and isn't needed — Navidrome starts with zero required env vars). Would have caused a `CreateContainerConfigError` on apply. Removed once it was clear no secret was necessary.

**4. Same cross-namespace Service addressing pitfall as Vaultwarden**
The new Cloudflare ingress rule needed the fully-qualified in-cluster DNS name (`http://navidrome.navidrome.svc.cluster.local:4533`), not the short form that only works for services in `cloudflared`'s own namespace.

**5. ConfigMap edits still didn't trigger a pod restart**
Same as the Vaultwarden setup — updating the shared `cloudflared` ConfigMap doesn't restart already-running pods. Confirmed the ConfigMap had the new rule via `kubectl get configmap cloudflared -n linkding -o yaml`, then forced `kubectl rollout restart deployment cloudflared -n linkding` to pick it up.

### Result
Navidrome is reachable at `navidrome.tonygunnca.com`, serving a personal library from an external drive mounted directly on the host, with app data persisted via PVC — confirmed working end-to-end including a live library scan and playback.