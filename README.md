# Olares Qwen3-Coder (Ollama)

This repository contains Olares app configurations for **Qwen3-Coder** served via **Ollama** — a powerful coding-focused large language model from Alibaba, optimized for GPU inference on 24GB VRAM.

---

## Apps

| App folder | Model | Quantization |
|------------|-------|--------------|
| [`ollamaqwen3coder32bq4km`](./ollamaqwen3coder32bq4km) | `qwen3-coder:32b-q4_K_M` | Q4_K_M |

---

## How It Works

### Shared App Pattern

Each app uses the Olares **shared app** pattern:

- The **admin** installs the app once. This spins up the Ollama server and model storage on the node.
- **Other users** install a lightweight reference app (no GPU resources) that proxies requests through to the admin's running instance.
- This means the 32B model is loaded once on the GPU and shared across all users in the cluster.

### Model Download

Ollama downloads the model **automatically on first use**. There is no separate download step:

1. On first request, the `harveyff-olares-ollama` API sidecar sends a pull request to the Ollama server (`http://localhost:11434`).
2. Ollama streams the model from the [Ollama Library](https://ollama.com/library/qwen3-coder) and stores it in the persistent volume at `~/Ollama/<release-name>` (mapped to `/root/.ollama` inside the container).
3. Subsequent requests are served directly from the cached model on disk.

The model directory is a `hostPath` volume so the model survives pod restarts without re-downloading.

### GPU Acceleration

The deployment uses the `applications.app.bytetrade.io/gpu-inject: "true"` annotation which causes Olares to inject the node GPU into the pod. The environment variable `GGML_CUDA_DISABLE_GRAPHS=1` is set to work around a known CUDA graph issue with some GPU drivers.

### Request Flow

```
User request
    → nginx (port 8080)               # ollamaclient Service
        → Ollama API sidecar (8081)   # harveyff-olares-ollama
            → Ollama server (11434)   # beclab/ollama-ollama
```

For reference (non-admin) users the nginx proxy forwards to the admin's namespace:

```
http://api.<release-name>-<admin-username>:8081
```

---

## App Structure

Each app follows the standard Olares Helm chart structure, identical to other Qwen apps in [beclab/apps](https://github.com/beclab/apps):

```
<appname>/
├── Chart.yaml                       # Helm chart metadata (name, appVersion = model tag)
├── OlaresManifest.yaml              # Olares app manifest (resources, permissions, entrances)
├── values.yaml                      # Empty — all values are injected by the Olares system
├── owners                           # List of GitHub usernames who maintain this app
├── templates/
│   ├── deployment.yaml              # Ollama server + API sidecar (admin-only, guarded by if-admin block)
│   └── clientproxy.yaml             # nginx ConfigMap + Deployment + Service (all users)
└── i18n/
    └── en-US/
        └── OlaresManifest.yaml      # English locale overrides for title/description
```

### `Chart.yaml`

Standard Helm v2 chart. The `appVersion` field holds the full model tag (e.g. `qwen3-coder:32b-q4_K_M`).

### `OlaresManifest.yaml`

The manifest is a Helm template. It uses a conditional block to apply different resource limits depending on whether the current user is the admin:

```yaml
{{- if and .Values.admin .Values.bfl.username (eq .Values.admin .Values.bfl.username) }}
# Admin: full GPU + memory resources
requiredGpu: 1Gi
limitedGpu: 24Gi
requiredMemory: 4200Mi
limitedMemory: 40Gi
{{- else }}
# Reference user: minimal resources (nginx proxy only)
requiredMemory: 64Mi
limitedMemory: 800Mi
{{- end }}
```

The `appScope` section marks the admin install as `clusterScoped: true` with an `appRef` listing the app name, so reference installs can declare the admin app as a mandatory dependency.

### `templates/deployment.yaml`

The entire Deployment and Service for the Ollama stack is wrapped in the `if-admin` guard, so it is only created in the admin's namespace:

```yaml
{{- if and .Values.admin .Values.bfl.username (eq .Values.admin .Values.bfl.username) }}
...
{{- end }}
```

The pod runs two containers:

| Container | Image | Role |
|-----------|-------|------|
| `ollama` | `beclab/ollama-ollama:0.17.5` | Ollama inference server on port 11434 |
| `api` | `beclab/harveyff-olares-ollama:v0.1.9` | Olares API sidecar on port 8080 (exposed as 8081 via Service) |

The `api` sidecar handles model pull on first use, health reporting, and proxying OpenAI-compatible requests to the Ollama server.

The persistent volume for model storage:

```yaml
volumes:
  - name: data
    hostPath:
      path: "{{ .Values.userspace.userData }}/Ollama/{{ .Release.Name }}"
      type: DirectoryOrCreate
```

### `templates/clientproxy.yaml`

Deployed for **all users** (admin and reference). Contains:

- A `ConfigMap` with the nginx configuration that routes to either the local `api` service (admin) or the cross-namespace `api.<release-name>-<admin>` service (reference user).
- A `Deployment` running the `beclab/aboveos-bitnami-openresty` nginx image.
- A `Service` named `ollamaclient` on port 8080 — this is what the `entrances` section in the manifest points to.

---

## Prerequisites

- Olares ≥ 1.12.2
- A GPU node with ≥ 1GB VRAM (24GB VRAM recommended for full 32B model performance)
- ≥ 4.2GB system RAM on the GPU node

---

## Adding a New Ollama App

To add another model (e.g. a different quantization or a different Qwen3-Coder variant):

1. Copy the existing app folder and rename it to match the model tag exactly (lowercase, no special characters except letters and numbers), e.g. `ollamaqwen3coder14bq4km`.
2. Update `Chart.yaml`: set `name` and `appVersion` to the new model tag.
3. Update `OlaresManifest.yaml`: set `metadata.name`, `metadata.appid`, `spec.versionName`, and all `appRef`/dependency `name` fields to the new app name.
4. Update `templates/deployment.yaml`: set `OLLAMA_MODEL` env var to the new model tag and update all `io.kompose.service` labels to the new app name.
5. Update `templates/clientproxy.yaml`: update the cross-namespace proxy URL if the service name changed.
6. Update `owners` with your GitHub username.
7. Add an entry to `i18n/en-US/OlaresManifest.yaml` with the English title and description.
