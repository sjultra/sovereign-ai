# Sovereign AI

Helm chart for a local LLM: [Ollama](https://ollama.com) on an NVIDIA GPU with [Open WebUI](https://github.com/open-webui/open-webui) as the chat interface. Installed by the Portal platform through a Fleet `HelmOp`, which passes placement and model settings as values at install time.

## Repo structure

```
service/
  Chart.yaml              # chart name (sovereign-ai-chart) and version
  values.yaml             # defaults; every key is documented inline
  templates/
    ollama.yaml           # Deployment, Service, PVC (only when models.source=pvc)
    openwebui.yaml        # Deployment, NodePort Service, Ingress
```

## Cluster requirements

- A node with an NVIDIA GPU, with the driver and NVIDIA Container Toolkit installed, so k3s registers the `nvidia` runtime.
- The NVIDIA device plugin, so the node advertises `nvidia.com/gpu`.
- The `local-path` StorageClass (k3s default), when `ollama.models.source=pvc`.
- Images are multi-arch (amd64 and arm64).

## Models

`ollama.models.source` decides where models live:

| `source` | Volume | Use |
|---|---|---|
| `pvc` (default) | `local-path` PVC (`persistence.size`) | Models are downloaded on demand. |
| `hostPath` | `models.hostPath` on the node, `type: Directory` | Models preloaded on the node with `ollama pull`, for fast demos. Pin the pod to that node. If the directory is missing, the pod stays in `ContainerCreating`. |

- Switching an install from `pvc` to `hostPath` deletes the PVC and the models downloaded into it.
- `ollama.preload` lists models to load into GPU memory before the pod is Ready.
- `keepAlive: "-1"` keeps loaded models in GPU memory.

## Placement

The chart sets no placement by default. The caller passes `nodeSelector`, `affinity` and `tolerations` for each component:

```bash
helm template t service \
  --set 'ollama.nodeSelector.kubernetes\.io/hostname=<node>' \
  --set 'openwebui.nodeSelector.kubernetes\.io/hostname=<node>'
```

Ollama uses `strategy: Recreate`, because its single GPU and RWO volume can't be shared with a replacement pod during a rollout.

## Updating

1. Edit `service/`.
2. Check it:
   ```bash
   helm lint service
   helm template t service
   helm template t service --set ollama.models.source=hostPath
   ```
3. Bump `version` in `service/Chart.yaml`. Published versions are never overwritten, because installs reference an exact version.
4. When bumping image tags, keep them pinned (never `latest`) and check that they have arm64 builds:
   ```bash
   docker manifest inspect ollama/ollama:<tag> | grep arm64
   ```

## Publishing

Charts are published to GHCR as OCI artifacts at `oci://ghcr.io/sjultra/charts/sovereign-ai-chart`. Log in with a classic personal access token that has the `write:packages` scope, authorized for SSO if the org requires it:

```bash
echo $GHCR_PAT | helm registry login ghcr.io -u <github-user> --password-stdin
helm package service
helm push sovereign-ai-chart-<version>.tgz oci://ghcr.io/sjultra/charts
```

## Manual install (testing)

```bash
helm install sovereign-ai oci://ghcr.io/sjultra/charts/sovereign-ai-chart --version <version> \
  --namespace sovereign-ai --create-namespace
```

## Known issues

- The ingress host is hard-coded in `values.yaml`.
- NodePort `30443` is hard-coded in `openwebui.yaml`. Fixed NodePorts allow only one install per cluster.
- Open WebUI has no persistent volume, so chats and settings are lost when the pod restarts.
