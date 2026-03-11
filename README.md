# Olares Qwen3-Coder App

This repository contains the Olares app configuration for **Qwen3-Coder 32B (vLLM)** — a powerful coding-focused large language model from Alibaba's Qwen3 series, optimized for GPU inference on 24GB VRAM.

## App: `vllmqwen3coder32binstruct4bit`

| Property | Value |
|----------|-------|
| Model | `Qwen/Qwen3-Coder-32B-Instruct-AWQ` |
| Quantization | AWQ 4-bit |
| Serving Framework | vLLM |
| Required GPU VRAM | ≥ 16Gi |
| Max GPU VRAM | 30Gi (optimized for 24GB) |
| Required RAM | ≥ 22Gi |
| Source | [HuggingFace](https://huggingface.co/Qwen/Qwen3-Coder-32B-Instruct-AWQ) |

## Structure

The app follows the standard Olares app structure identical to other Qwen apps in [beclab/apps](https://github.com/beclab/apps):

```
vllmqwen3coder32binstruct4bit/
├── Chart.yaml                    # Helm chart metadata
├── OlaresManifest.yaml          # Olares app configuration with GPU resources
├── values.yaml                  # (empty - values come from Olares system)
├── owners                       # App maintainers
├── templates/
│   ├── deployment.yaml          # vLLM server + HuggingFace model downloader sidecar
│   └── clientproxy.yaml         # nginx reverse proxy for OpenAI-compatible API
└── i18n/
    ├── en-US/OlaresManifest.yaml
    └── zh-CN/OlaresManifest.yaml
```

## How It Works

1. **Shared App**: The admin installs the app once; all cluster users can access it via a reference app.
2. **Model Download**: The `harveyff-hf-downloader` sidecar downloads `Qwen/Qwen3-Coder-32B-Instruct-AWQ` from HuggingFace to persistent storage under `~/Huggingface/qwen3coder32b`.
3. **GPU Inference**: Once downloaded, vLLM serves the model using AWQ quantization with GPU acceleration (`gpu-inject: "true"` annotation).
4. **API Access**: An nginx proxy exposes the OpenAI-compatible API on port 8080, accessible to cluster users.

## Prerequisites

- Olares ≥ 1.12.2
- GPU with ≥ 16GB VRAM (24GB recommended)
- HuggingFace account with token and access to the model
- ≥ 22GB system RAM
