## Vaultwarden

Self-hosted password manager (Bitwarden-compatible server), deployed to the `pi-cluster` via Flux/Kustomize and exposed through a shared Cloudflare Tunnel.

### Architecture

- **Base manifests** (`apps/base/vaultwarden/`): Namespace, PersistentVolumeClaim, Deployment, and Service — environment-agnostic.
- **Staging overlay** (`apps/staging/vaultwarden/`): applies the `vaultwarden` namespace via Kustomize, and layers in an environment-specific, SOPS-encrypted Secret (`ADMIN_TOKEN`, `DOMAIN`, `SIGNUPS_ALLOWED`, `WEBSOCKET_ENABLED`).
- **Ingress**: routed through the cluster's existing shared `cloudflared` tunnel (already running for Linkding) rather than a dedicated tunnel per app — one more `ingress` rule added to the shared ConfigMap, pointing at Vaultwarden's Service using its fully-qualified cross-namespace address.
- **Reconciliation**: Flux watches `apps/staging/` as a whole; no manual Flux Kustomization CR was needed — dropping a new app folder in was enough for it to be picked up automatically.

### Notable issues hit during setup (and how they were diagnosed)

**1. SOPS config scope was too narrow**
The repo's `.sops.yaml` lived under `clusters/staging/`, which only covers files in that directory tree by upward directory search — it didn't reach `apps/staging/vaultwarden/`. Confirmed by locating the file with `find`, then moved `.sops.yaml` to the repo root so its `path_regex` rule applies repo-wide, covering both `clusters/` and `apps/`.

**2. Wrong container port**
The Deployment and Service were configured for port `8080`, but Vaultwarden's own logs showed `Rocket has launched from http://0.0.0.0:80` — the image listens on port 80 by default. Caught by checking `kubectl logs` before assuming the Service config was correct, then aligned `containerPort`/`targetPort` to 80.

**3. Cross-namespace Service addressing in the Cloudflare ingress rule**
The new ingress rule initially pointed at `http://vaultwarden:80` — which works for Linkding only because `cloudflared` and Linkding's Service share the same namespace. Vaultwarden's Service lives in its own `vaultwarden` namespace, so the short name didn't resolve. Fixed by using the fully-qualified in-cluster DNS name: `http://vaultwarden.vaultwarden.svc.cluster.local:80`.

**4. ConfigMap edits don't trigger a pod restart**
Updating the shared `cloudflared` ConfigMap doesn't restart the pods that already loaded it into memory — they kept running with the old ingress rules until manually restarted with `kubectl rollout restart deployment cloudflared -n linkding`.

**5. A duplicated YAML key broke the tunnel config entirely**
A copy-paste edit left `service: service: http://...` in the ConfigMap — an easy typo, but it caused a hard YAML parse failure (`mapping values are not allowed in this context`), which crash-looped both `cloudflared` replicas and surfaced as a Cloudflare Error 1033 (no active tunnel connection). Diagnosed by reading the pod logs directly rather than assuming the DNS/tunnel setup itself was at fault.

### Result

Vaultwarden is reachable at `vault.tonygunnca.com`, backed by persistent storage, with secrets encrypted at rest via SOPS/age and applied automatically by Flux — no manual `kubectl apply` steps in the deploy path.
