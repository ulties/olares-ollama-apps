# Olares Qwen3-Coder Apps

This repository contains Olares app configurations for **Qwen3-Coder** — a powerful coding-focused large language model from Alibaba, optimized for GPU inference on 24GB VRAM.

Two serving variants are provided:

---

## App: `vllmqwen3coder32binstructawq` (vLLM)

| Property | Value |
|----------|-------|
| Model | `Qwen/Qwen3-Coder-32B-Instruct-AWQ` |
| Quantization | AWQ 4-bit |
| Serving Framework | vLLM |
| Required GPU VRAM | ≥ 16Gi |
| Max GPU VRAM | 30Gi (optimized for 24GB) |
| Required RAM | ≥ 22Gi |
| Source | [HuggingFace](https://huggingface.co/Qwen/Qwen3-Coder-32B-Instruct-AWQ) |

### How It Works

1. **Shared App**: The admin installs the app once; all cluster users can access it via a reference app.
2. **Model Download**: The `harveyff-hf-downloader` sidecar downloads the model from HuggingFace to persistent storage under `~/Huggingface/qwen3coder32b`.
3. **GPU Inference**: Once downloaded, vLLM serves the model using AWQ quantization with GPU acceleration (`gpu-inject: "true"` annotation).
4. **API Access**: An nginx proxy exposes the OpenAI-compatible API on port 8080, accessible to cluster users.

---

## App: `ollamaqwen3coder32bq4km` (Ollama)

| Property | Value |
|----------|-------|
| Model | `qwen3-coder:32b-q4_K_M` |
| Quantization | Q4_K_M |
| Serving Framework | Ollama |
| Required GPU VRAM | ≥ 1Gi |
| Max GPU VRAM | 24Gi (optimized for 24GB) |
| Required RAM | ≥ 4.2Gi |
| Source | [Ollama Library](https://ollama.com/library/qwen3-coder) |

### How It Works

1. **Shared App**: The admin installs the app once; all cluster users can access it via a reference app.
2. **Model Download**: Ollama automatically downloads `qwen3-coder:32b-q4_K_M` on first use, stored under `~/Ollama/<release-name>`.
3. **GPU Inference**: The Ollama server uses GPU acceleration via the `gpu-inject: "true"` annotation with `GGML_CUDA_DISABLE_GRAPHS=1` for optimized CUDA performance.
4. **API Access**: An nginx proxy on port 8080 routes requests to the Ollama API sidecar.

---

## Structure

Both apps follow the standard Olares app structure identical to other Qwen apps in [beclab/apps](https://github.com/beclab/apps):

```
<appname>/
├── Chart.yaml                    # Helm chart metadata
├── OlaresManifest.yaml          # Olares app configuration with GPU resources
├── values.yaml                  # (empty - values come from Olares system)
├── owners                       # App maintainers
├── templates/
│   ├── deployment.yaml          # Model server deployment
│   └── clientproxy.yaml         # nginx reverse proxy
└── i18n/
    └── en-US/OlaresManifest.yaml
```

## Prerequisites

- Olares ≥ 1.12.2
- GPU with ≥ 1GB VRAM (24GB recommended for full performance)
- For vLLM variant: HuggingFace account with token and ≥ 22GB system RAM
- For Ollama variant: ≥ 4.2GB system RAM
