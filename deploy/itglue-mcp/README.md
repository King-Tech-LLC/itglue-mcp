# itglue-mcp — k3s deployment (KingTech / kt-icg-agent P1)

Vendored fork of `wyre-technology/itglue-mcp`, deployed read-only on the KingTech
k3s cluster (tailnet) and consumed by the `kt-icg-agent` plugin at
`https://itglue-mcp.tail3cab53.ts.net/mcp`.

Deploy pattern mirrors `toggl-mcp/deploy/`. Manifests here are declarative; the
IT Glue API key is injected out-of-band as a Secret (never committed).

## Vendoring status

- Upstream: `wyre-technology/itglue-mcp` (Apache-2.0). Reviewed at commit
  `c52c750` (v1.5.3) — verdict SAFE-TO-VENDOR: single prod dependency (official
  MCP SDK), no telemetry, egress hard-scoped to IT Glue hosts, no password
  writes, non-root image, stateless. `get_password` returns credential values by
  design (a read). See KIA-5.
- Image PINNED to digest
  `ghcr.io/wyre-technology/itglue-mcp@sha256:dfd476ce8def3ce12a7a1eaf9f4850942b65f1c31b3d4118781fef880280689b`.
- Follow-up: mirror the digest into `harbor.tail3cab53.ts.net/apps/itglue-mcp`
  and repoint the Deployment (removes the ghcr runtime dependency).

## Prerequisites

- `KUBECONFIG=~/.kube/k3s-lab.yaml` (cluster over the tailnet).
- Tailscale Kubernetes operator + MagicDNS/HTTPS on `tail3cab53.ts.net` (already
  provisioned).
- A **read-only** IT Glue API key (`ITG.xxx`) — layer 2 of the read-first model.

## Deploy

```sh
export KUBECONFIG=~/.kube/k3s-lab.yaml

# 1. Namespace first.
kubectl apply -f deploy/itglue-mcp/namespace.yaml

# 2. Secret (out of band — never commit it). Create BEFORE the Deployment so the
#    pod doesn't CrashLoop waiting for it.
kubectl -n itglue-mcp create secret generic itglue-mcp-itglue \
  --from-literal=ITGLUE_API_KEY=ITG.xxxxxxxxxxxxxxxx

# 3. The rest.
kubectl apply -f deploy/itglue-mcp/

# 4. Wait + verify.
kubectl -n itglue-mcp rollout status deploy/itglue-mcp
curl -fsS https://itglue-mcp.tail3cab53.ts.net/health   # -> OK (allow ~30-60s for the ingress device + LE cert)
```

## Rolling the image forward

Resolve the new digest, then bump `image:` in `deployment.yaml`:

```sh
skopeo inspect --format '{{.Digest}}' docker://ghcr.io/wyre-technology/itglue-mcp:latest
# or: docker buildx imagetools inspect ghcr.io/wyre-technology/itglue-mcp:latest
kubectl apply -f deploy/itglue-mcp/deployment.yaml
```

## Uninstall

```sh
kubectl delete ns itglue-mcp
```

## Tool surface (for the plugin write-gate allowlist)

**Reads (allow, 15):** search_organizations, get_organization,
search_configurations, get_configuration, search_locations, get_location,
search_passwords, get_password, search_documents, get_document,
list_document_folders, list_document_sections, list_flexible_asset_types,
search_flexible_assets, itglue_health_check.

**Writes (ask/gated, 9):** create_location, update_location, create_document,
create_document_section, update_document_section, delete_document_section,
publish_document, archive_document, unarchive_document.
