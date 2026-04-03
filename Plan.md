# OME Platform — Complete Model Deployment Guide

> **Server:** RHEL 9.7, 128 cores, 1TB RAM, NVIDIA H100 (95,830 MB VRAM)
> **Container Runtime:** Podman (rootless or rootful)
> **Projects:** OME RAG (5 CPU models) + OME Agent (1 GPU model) = 6 llama-server instances
> **Goal:** All models running, healthy, and auto-restarting on boot

-----

## Table of Contents

1. [Architecture — 6 Models, 2 Projects](#1-architecture)
1. [Prerequisites — RHEL 9.7 Setup](#2-prerequisites)
1. [Directory Structure](#3-directory-structure)
1. [Model Downloads](#4-model-downloads)
1. [Build llama.cpp (CPU + CUDA)](#5-build-llamacpp)
1. [OME RAG Models — 5 CPU Instances (:8081-:8085)](#6-ome-rag-models)
1. [OME Agent Model — 1 GPU Instance (:8090)](#7-ome-agent-model)
1. [Podman Deployment — All 6 Models](#8-podman-deployment)
1. [Systemd Integration — Auto-Start on Boot](#9-systemd-integration)
1. [Health Checks & Monitoring](#10-health-checks)
1. [Startup & Shutdown Procedures](#11-startup-shutdown)
1. [Troubleshooting](#12-troubleshooting)
1. [Resource Allocation Summary](#13-resource-allocation)

-----

## 1. Architecture — 6 Models, 2 Projects

```
┌─────────────────────────────────────────────────────────────────────┐
│  RHEL 9.7 — 128 cores / 1TB RAM / H100 95,830 MB                   │
│                                                                     │
│  OME RAG (CPU models — existing, unchanged):                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  :8081  Snowflake Arctic Embed L v2.0  (Q8_0, 1024d)  CPU   │   │
│  │         Purpose: embed() — prose embedding                   │   │
│  │         Used by: Confluence, PR summaries, query embedding   │   │
│  │                                                              │   │
│  │  :8082  Qwen2.5-Coder-7B-Instruct     (Q4_K_M)        CPU   │   │
│  │         Purpose: generate() / chat() — contextual headers    │   │
│  │         Used by: contextual_enrich, summarize, pr_condense   │   │
│  │                                                              │   │
│  │  :8083  BGE Reranker v2-m3             (Q6_K)          CPU   │   │
│  │         Purpose: rerank() — cross-encoder reranking          │   │
│  │         Used by: hybrid_search stage 14 (20→5)               │   │
│  │                                                              │   │
│  │  :8084  Qwen2.5-Coder-1.5B-Instruct   (Q8_0)          CPU   │   │
│  │         Purpose: fast chat() — query_expand, fast tasks      │   │
│  │         Used by: query_expand, fast search-time LLM          │   │
│  │                                                              │   │
│  │  :8085  nomic-embed-code               (Q8_0, 3584d)  CPU   │   │
│  │         Purpose: embedCode() — code embedding                │   │
│  │         Used by: all Bitbucket source file embeddings        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  OME Agent (GPU model — NEW):                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  :8090  Qwen3-235B-A22B-Instruct-2507  (UD-Q4_K_XL)   GPU   │   │
│  │         Purpose: Agentic code generation                     │   │
│  │         VRAM: ~35 GB (non-MoE layers on GPU)                 │   │
│  │         RAM:  ~85 GB (MoE expert layers on CPU)              │   │
│  │         Context: 64K × 2 parallel slots                      │   │
│  │         Speed: ~15-25 tok/s                                  │   │
│  │         Used by: ome_rag_guide (UNDERSTAND, IMPLEMENT,       │   │
│  │                  DIAGNOSE modes)                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

-----

## 2. Prerequisites — RHEL 9.7 Setup

### 2.1 NVIDIA Driver + Container Toolkit

```bash
# Verify NVIDIA driver is installed
nvidia-smi
# Should show: H100, 95830 MiB

# Install NVIDIA Container Toolkit (for Podman GPU passthrough)
sudo dnf config-manager --add-repo \
  https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo

sudo dnf install -y nvidia-container-toolkit

# Generate CDI spec (Container Device Interface)
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Configure Podman to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=podman

# Verify GPU is accessible from containers
podman run --rm --device nvidia.com/gpu=all \
  nvidia/cuda:12.6.0-base-ubi9 nvidia-smi
```

### 2.2 Podman

```bash
# Should already be on RHEL 9.7
podman --version
# If not:
sudo dnf install -y podman podman-compose

# Enable lingering for rootless (if running as non-root user)
sudo loginctl enable-linger $(whoami)
```

### 2.3 Build Tools (For Building llama.cpp Locally — Alternative to Container)

```bash
sudo dnf install -y gcc gcc-c++ cmake make git curl \
  libcurl-devel openssl-devel
```

-----

## 3. Directory Structure

```bash
# Create the directory structure
sudo mkdir -p /opt/ome-platform/{rag,agent}
sudo mkdir -p /models/{arctic-embed,qwen25-7b,bge-reranker,qwen25-1.5b,nomic-embed,qwen3-235b-2507}
sudo mkdir -p /opt/ome-platform/llama.cpp
sudo mkdir -p /var/log/ome-platform

# Set ownership (adjust user as needed)
sudo chown -R ome:ome /opt/ome-platform /models /var/log/ome-platform
```

```
/opt/ome-platform/
├── rag/                          # OME RAG project (existing)
│   ├── dist/                     # Compiled TypeScript
│   ├── src/
│   └── package.json
├── agent/                        # OME Agent project (NEW)
│   ├── dist/
│   ├── src/
│   └── package.json
└── llama.cpp/                    # Built llama.cpp (if not using container)
    └── build/bin/
        ├── llama-server
        └── llama-cli

/models/
├── arctic-embed/                 # :8081 Snowflake Arctic Embed L v2.0
│   └── snowflake-arctic-embed-l-v2.0-Q8_0.gguf
├── qwen25-7b/                    # :8082 Qwen2.5-Coder-7B
│   └── qwen2.5-coder-7b-instruct-Q4_K_M.gguf
├── bge-reranker/                 # :8083 BGE Reranker v2-m3
│   └── bge-reranker-v2-m3-Q6_K.gguf
├── qwen25-1.5b/                  # :8084 Qwen2.5-Coder-1.5B
│   └── qwen2.5-coder-1.5b-instruct-Q8_0.gguf
├── nomic-embed/                  # :8085 nomic-embed-code
│   └── nomic-embed-code-Q8_0.gguf
└── qwen3-235b-2507/              # :8090 Qwen3-235B-A22B (NEW)
    └── UD-Q4_K_XL/
        ├── Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf  (49.9 GB)
        ├── Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00002-of-00003.gguf  (49.8 GB)
        └── Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00003-of-00003.gguf  (34.6 GB)
```

-----

## 4. Model Downloads

### 4.1 Existing RAG Models (Skip If Already Downloaded)

These should already be on your server. Verify:

```bash
ls -lh /models/arctic-embed/*.gguf        # ~1.2 GB
ls -lh /models/qwen25-7b/*.gguf           # ~4.1 GB
ls -lh /models/bge-reranker/*.gguf         # ~1.1 GB
ls -lh /models/qwen25-1.5b/*.gguf         # ~1.6 GB
ls -lh /models/nomic-embed/*.gguf          # ~0.5 GB (or larger depending on quant)
```

### 4.2 New Agent Model — Qwen3-235B-A22B-Instruct-2507

Download on a Windows machine (or any machine with internet), then transfer:

```powershell
# On Windows (PowerShell) — using aria2c for reliability
$base = "https://huggingface.co/unsloth/Qwen3-235B-A22B-Instruct-2507-GGUF/resolve/main/UD-Q4_K_XL"
$dir = "C:\Models\qwen3-235b-2507\UD-Q4_K_XL"
mkdir -Force $dir

aria2c -x 16 -s 16 --retry-wait=30 --max-tries=10 --timeout=600 `
  -d $dir "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf"

aria2c -x 16 -s 16 --retry-wait=30 --max-tries=10 --timeout=600 `
  -d $dir "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00002-of-00003.gguf"

aria2c -x 16 -s 16 --retry-wait=30 --max-tries=10 --timeout=600 `
  -d $dir "$base/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00003-of-00003.gguf"
```

Transfer to RHEL server:

```bash
# From Windows (using scp or rsync via WSL)
scp C:\Models\qwen3-235b-2507\UD-Q4_K_XL\*.gguf \
  ome@rhel-server:/models/qwen3-235b-2507/UD-Q4_K_XL/
```

Verify on RHEL:

```bash
ls -lh /models/qwen3-235b-2507/UD-Q4_K_XL/
# Expected:
#   -00001-of-00003.gguf   49.9 GB
#   -00002-of-00003.gguf   49.8 GB
#   -00003-of-00003.gguf   34.6 GB
#   Total:                 134.3 GB
```

-----

## 5. Build llama.cpp (CPU + CUDA)

You need **TWO builds** — CPU-only (for RAG models) and CUDA (for Agent model). Or use the official container image for the GPU model.

### Option A: Build Both Locally

```bash
cd /opt/ome-platform

# Clone llama.cpp
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

# ── Build 1: CPU-only (for RAG models :8081-:8085) ──
cmake -B build-cpu \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_CUDA=OFF \
  -DLLAMA_CURL=ON
cmake --build build-cpu --config Release -j$(nproc) --target llama-server
cp build-cpu/bin/llama-server /opt/ome-platform/llama-server-cpu

# ── Build 2: CUDA (for Agent model :8090) ──
cmake -B build-cuda \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_CUDA=ON \
  -DLLAMA_CURL=ON \
  -DGGML_CUDA_FA_ALL_QUANTS=ON
cmake --build build-cuda --config Release -j$(nproc) --target llama-server
cp build-cuda/bin/llama-server /opt/ome-platform/llama-server-cuda
```

### Option B: Use Container for GPU, Local Build for CPU

```bash
# Build CPU-only locally (for RAG models)
# Same as Build 1 above

# Pull container for GPU model (no build needed)
podman pull ghcr.io/ggml-org/llama.cpp:server-cuda
```

-----

## 6. OME RAG Models — 5 CPU Instances (:8081-:8085)

These are your **existing** models. Commands documented for completeness and standardization.

### 6.1 Port :8081 — Snowflake Arctic Embed L v2.0 (Prose Embedding)

```bash
/opt/ome-platform/llama-server-cpu \
  --model /models/arctic-embed/snowflake-arctic-embed-l-v2.0-Q8_0.gguf \
  --host 127.0.0.1 \
  --port 8081 \
  --embedding \
  --ctx-size 512 \
  --batch-size 512 \
  --threads 32 \
  --parallel 4
```

|Detail     |Value                                                      |
|-----------|-----------------------------------------------------------|
|Purpose    |Prose embedding (Confluence, PR summaries, query embedding)|
|Dimensions |1024d                                                      |
|Format     |Q8_0 GGUF                                                  |
|CPU threads|32                                                         |
|Context    |512 (embedding model — short inputs)                       |

### 6.2 Port :8082 — Qwen2.5-Coder-7B-Instruct (LLM Generator)

```bash
/opt/ome-platform/llama-server-cpu \
  --model /models/qwen25-7b/qwen2.5-coder-7b-instruct-Q4_K_M.gguf \
  --host 127.0.0.1 \
  --port 8082 \
  --ctx-size 8192 \
  --threads 16 \
  --parallel 2 \
  --jinja
```

|Detail     |Value                                                   |
|-----------|--------------------------------------------------------|
|Purpose    |contextual_enrich, summarize, pr_condense, jira_condense|
|Format     |Q4_K_M GGUF                                             |
|CPU threads|16                                                      |
|Context    |8192                                                    |

### 6.3 Port :8083 — BGE Reranker v2-m3 (Cross-Encoder Reranking)

```bash
/opt/ome-platform/llama-server-cpu \
  --model /models/bge-reranker/bge-reranker-v2-m3-Q6_K.gguf \
  --host 127.0.0.1 \
  --port 8083 \
  --reranking \
  --ctx-size 512 \
  --batch-size 512 \
  --threads 8 \
  --parallel 1
```

|Detail     |Value                                                   |
|-----------|--------------------------------------------------------|
|Purpose    |Cross-encoder reranking (20→5 in hybrid_search stage 14)|
|Format     |Q6_K GGUF                                               |
|CPU threads|8                                                       |
|Context    |512 (reranker — short query+doc pairs)                  |

### 6.4 Port :8084 — Qwen2.5-Coder-1.5B-Instruct (Fast LLM)

```bash
/opt/ome-platform/llama-server-cpu \
  --model /models/qwen25-1.5b/qwen2.5-coder-1.5b-instruct-Q8_0.gguf \
  --host 127.0.0.1 \
  --port 8084 \
  --ctx-size 4096 \
  --threads 8 \
  --parallel 2 \
  --jinja
```

|Detail     |Value                             |
|-----------|----------------------------------|
|Purpose    |query_expand, fast search-time LLM|
|Format     |Q8_0 GGUF                         |
|CPU threads|8                                 |
|Context    |4096                              |

### 6.5 Port :8085 — nomic-embed-code (Code Embedding)

```bash
/opt/ome-platform/llama-server-cpu \
  --model /models/nomic-embed/nomic-embed-code-Q8_0.gguf \
  --host 127.0.0.1 \
  --port 8085 \
  --embedding \
  --ctx-size 32768 \
  --batch-size 2048 \
  --threads 24 \
  --parallel 3
```

|Detail     |Value                                       |
|-----------|--------------------------------------------|
|Purpose    |Code embedding (all Bitbucket source files) |
|Dimensions |3584d                                       |
|Format     |Q8_0 GGUF                                   |
|CPU threads|24                                          |
|Context    |32768 (nomic supports long context for code)|

-----

## 7. OME Agent Model — 1 GPU Instance (:8090)

### 7.1 Native (If Built Locally)

```bash
/opt/ome-platform/llama-server-cuda \
  --model /models/qwen3-235b-2507/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf \
  --host 127.0.0.1 \
  --port 8090 \
  --n-gpu-layers 99 \
  -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 65536 \
  --parallel 2 \
  --threads 32 \
  --flash-attn \
  --jinja \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.0
```

### 7.2 Podman Container (Recommended)

```bash
podman run -d \
  --name ome-agent-llm \
  --device nvidia.com/gpu=all \
  --restart always \
  --network host \
  --memory=900g \
  -v /models/qwen3-235b-2507:/models:Z \
  --health-cmd="curl -f http://localhost:8090/health || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  --health-start-period=180s \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf \
  --host 0.0.0.0 \
  --port 8090 \
  --n-gpu-layers 99 \
  -ot ".ffn_.*_exps.=CPU" \
  --ctx-size 65536 \
  --parallel 2 \
  --threads 32 \
  --flash-attn \
  --jinja \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.0
```

|Detail            |Value                                        |
|------------------|---------------------------------------------|
|Model             |Qwen3-235B-A22B-Instruct-2507                |
|Quant             |UD-Q4_K_XL (Unsloth Dynamic 2.0)             |
|Total size on disk|134.3 GB (3 split GGUF files)                |
|VRAM used         |~35 GB (non-MoE layers on GPU)               |
|CPU RAM used      |~85 GB (MoE expert layers offloaded)         |
|KV cache VRAM     |~50 GB (2 slots × 64K context)               |
|Total VRAM        |~85 GB / 93.6 GB available                   |
|Headroom          |~9 GB                                        |
|Speed             |~15-25 tok/s                                 |
|Context           |65536 per slot (holds 8-10 full source files)|
|Parallel slots    |2 (two developers simultaneously)            |

**Critical flags:**

- `--n-gpu-layers 99` — put all possible layers on GPU
- `-ot ".ffn_.*_exps.=CPU"` — offload MoE expert layers to CPU RAM
- `--threads 32` — CPU threads for expert computation
- `--flash-attn` — H100 native flash attention
- `--jinja` — required for Qwen3 chat template + tool calling
- `--parallel 2` — two concurrent inference slots

-----

## 8. Podman Deployment — All 6 Models

### 8.1 Single Compose File

```yaml
# /opt/ome-platform/podman-compose.yml

version: "3"

services:

  # ═══ OME RAG Models (CPU) ═══

  arctic-embed:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: ome-arctic-embed
    restart: always
    network_mode: host
    volumes:
      - /models/arctic-embed:/models:Z
    command: >
      --model /models/snowflake-arctic-embed-l-v2.0-Q8_0.gguf
      --host 0.0.0.0 --port 8081
      --embedding --ctx-size 512 --batch-size 512
      --threads 32 --parallel 4
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  qwen25-7b:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: ome-qwen25-7b
    restart: always
    network_mode: host
    volumes:
      - /models/qwen25-7b:/models:Z
    command: >
      --model /models/qwen2.5-coder-7b-instruct-Q4_K_M.gguf
      --host 0.0.0.0 --port 8082
      --ctx-size 8192 --threads 16 --parallel 2 --jinja
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  bge-reranker:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: ome-bge-reranker
    restart: always
    network_mode: host
    volumes:
      - /models/bge-reranker:/models:Z
    command: >
      --model /models/bge-reranker-v2-m3-Q6_K.gguf
      --host 0.0.0.0 --port 8083
      --reranking --ctx-size 512 --batch-size 512
      --threads 8 --parallel 1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8083/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  qwen25-1_5b:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: ome-qwen25-1.5b
    restart: always
    network_mode: host
    volumes:
      - /models/qwen25-1.5b:/models:Z
    command: >
      --model /models/qwen2.5-coder-1.5b-instruct-Q8_0.gguf
      --host 0.0.0.0 --port 8084
      --ctx-size 4096 --threads 8 --parallel 2 --jinja
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8084/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  nomic-embed:
    image: ghcr.io/ggml-org/llama.cpp:server
    container_name: ome-nomic-embed
    restart: always
    network_mode: host
    volumes:
      - /models/nomic-embed:/models:Z
    command: >
      --model /models/nomic-embed-code-Q8_0.gguf
      --host 0.0.0.0 --port 8085
      --embedding --ctx-size 32768 --batch-size 2048
      --threads 24 --parallel 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8085/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  # ═══ OME Agent Model (GPU) ═══

  qwen3-235b:
    image: ghcr.io/ggml-org/llama.cpp:server-cuda
    container_name: ome-qwen3-235b
    restart: always
    network_mode: host
    devices:
      - nvidia.com/gpu=all
    volumes:
      - /models/qwen3-235b-2507:/models:Z
    deploy:
      resources:
        limits:
          memory: 900g
    command: >
      --model /models/UD-Q4_K_XL/Qwen3-235B-A22B-Instruct-2507-UD-Q4_K_XL-00001-of-00003.gguf
      --host 0.0.0.0 --port 8090
      --n-gpu-layers 99
      -ot ".ffn_.*_exps.=CPU"
      --ctx-size 65536 --parallel 2
      --threads 32 --flash-attn --jinja
      --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 180s
```

### 8.2 Start Everything

```bash
cd /opt/ome-platform
podman-compose up -d

# Watch logs
podman-compose logs -f
```

### 8.3 Start Individually (If Not Using Compose)

```bash
# Start CPU models first (fast, <30s each)
podman start ome-arctic-embed
podman start ome-nomic-embed
podman start ome-bge-reranker
podman start ome-qwen25-7b
podman start ome-qwen25-1.5b

# Start GPU model last (slow, ~2-3 min for MoE loading)
podman start ome-qwen3-235b
```

-----

## 9. Systemd Integration — Auto-Start on Boot

### 9.1 Generate Systemd Units from Podman

```bash
# For each container, generate a systemd unit
mkdir -p ~/.config/systemd/user/

podman generate systemd --new --name ome-arctic-embed \
  > ~/.config/systemd/user/container-ome-arctic-embed.service

podman generate systemd --new --name ome-nomic-embed \
  > ~/.config/systemd/user/container-ome-nomic-embed.service

podman generate systemd --new --name ome-bge-reranker \
  > ~/.config/systemd/user/container-ome-bge-reranker.service

podman generate systemd --new --name ome-qwen25-7b \
  > ~/.config/systemd/user/container-ome-qwen25-7b.service

podman generate systemd --new --name ome-qwen25-1.5b \
  > ~/.config/systemd/user/container-ome-qwen25-1.5b.service

podman generate systemd --new --name ome-qwen3-235b \
  > ~/.config/systemd/user/container-ome-qwen3-235b.service

# Reload and enable all
systemctl --user daemon-reload

systemctl --user enable container-ome-arctic-embed.service
systemctl --user enable container-ome-nomic-embed.service
systemctl --user enable container-ome-bge-reranker.service
systemctl --user enable container-ome-qwen25-7b.service
systemctl --user enable container-ome-qwen25-1.5b.service
systemctl --user enable container-ome-qwen3-235b.service
```

### 9.2 Alternative: Single Systemd Service for Compose

```ini
# /etc/systemd/system/ome-platform.service
[Unit]
Description=OME Platform — All AI Models
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/ome-platform
ExecStart=/usr/bin/podman-compose up -d
ExecStop=/usr/bin/podman-compose down
User=ome
Group=ome

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable ome-platform.service
sudo systemctl start ome-platform.service
```

### 9.3 Startup Order Control

The GPU model takes 2-3 minutes to load. OME RAG and Agent Node.js processes should wait for their models:

```ini
# /etc/systemd/system/ome-rag.service
[Unit]
Description=OME RAG — Node.js MCP Server
After=ome-platform.service
Wants=ome-platform.service

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-platform/rag
ExecStartPre=/opt/ome-platform/scripts/wait-for-models.sh 8081 8082 8083 8084 8085
ExecStart=/usr/bin/node --experimental-strip-types dist/server/http_entry.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/ome-agent.service
[Unit]
Description=OME Agent — Node.js MCP Server
After=ome-platform.service ome-rag.service
Wants=ome-platform.service ome-rag.service

[Service]
Type=simple
User=ome
WorkingDirectory=/opt/ome-platform/agent
ExecStartPre=/opt/ome-platform/scripts/wait-for-models.sh 8090
ExecStart=/usr/bin/node --experimental-strip-types dist/server/http_entry.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

### 9.4 Wait-for-Models Script

```bash
#!/bin/bash
# /opt/ome-platform/scripts/wait-for-models.sh
# Waits for llama-server health endpoints to be ready

MAX_WAIT=300  # 5 minutes max
INTERVAL=5

for PORT in "$@"; do
  echo "Waiting for model on port $PORT..."
  WAITED=0
  while ! curl -sf "http://127.0.0.1:${PORT}/health" > /dev/null 2>&1; do
    sleep $INTERVAL
    WAITED=$((WAITED + INTERVAL))
    if [ $WAITED -ge $MAX_WAIT ]; then
      echo "ERROR: Model on port $PORT not ready after ${MAX_WAIT}s"
      exit 1
    fi
    echo "  Still waiting... (${WAITED}s)"
  done
  echo "Model on port $PORT is ready."
done

echo "All models ready."
```

```bash
chmod +x /opt/ome-platform/scripts/wait-for-models.sh
```

-----

## 10. Health Checks & Monitoring

### 10.1 Quick Health Check Script

```bash
#!/bin/bash
# /opt/ome-platform/scripts/health-check.sh

echo "═══ OME Platform Health Check ═══"
echo ""

PORTS=(8081 8082 8083 8084 8085 8090)
NAMES=(
  "Arctic Embed (prose)"
  "Qwen2.5-7B (LLM)"
  "BGE Reranker"
  "Qwen2.5-1.5B (fast)"
  "nomic-embed (code)"
  "Qwen3-235B (agent)"
)

ALL_OK=true

for i in "${!PORTS[@]}"; do
  PORT=${PORTS[$i]}
  NAME=${NAMES[$i]}

  STATUS=$(curl -sf "http://127.0.0.1:${PORT}/health" 2>/dev/null)
  if [ $? -eq 0 ]; then
    echo "  ✅  :${PORT}  ${NAME}"
  else
    echo "  ❌  :${PORT}  ${NAME}  — NOT RESPONDING"
    ALL_OK=false
  fi
done

echo ""

# GPU status
echo "═══ GPU Status ═══"
nvidia-smi --query-gpu=name,memory.used,memory.total,temperature.gpu,utilization.gpu \
  --format=csv,noheader,nounits 2>/dev/null | \
  awk -F', ' '{printf "  GPU: %s | VRAM: %s/%s MB | Temp: %s°C | Util: %s%%\n", $1, $2, $3, $4, $5}'

echo ""

# Container status
echo "═══ Container Status ═══"
podman ps --format "  {{.Names}}\t{{.Status}}" | grep ome-

echo ""

if [ "$ALL_OK" = true ]; then
  echo "✅ All models healthy."
  exit 0
else
  echo "⚠️  Some models are not responding."
  exit 1
fi
```

```bash
chmod +x /opt/ome-platform/scripts/health-check.sh
```

### 10.2 Cron-Based Monitoring

```bash
# Check every 5 minutes, log failures
echo "*/5 * * * * /opt/ome-platform/scripts/health-check.sh >> /var/log/ome-platform/health.log 2>&1" \
  | crontab -
```

### 10.3 Test Each Model

```bash
# ── Test embedding models ──
# Arctic Embed (:8081) — prose
curl -s http://127.0.0.1:8081/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "test embedding", "model": "arctic"}' | jq '.data[0].embedding | length'
# Expected: 1024

# nomic-embed-code (:8085) — code
curl -s http://127.0.0.1:8085/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "Represent this code snippet: function hello() { return 1; }", "model": "nomic"}' \
  | jq '.data[0].embedding | length'
# Expected: 3584

# ── Test reranker ──
curl -s http://127.0.0.1:8083/v1/rerank \
  -H "Content-Type: application/json" \
  -d '{"query": "hello", "documents": ["hello world", "goodbye world"], "model": "bge"}' \
  | jq '.results'

# ── Test LLM models ──
# Qwen2.5-7B (:8082)
curl -s http://127.0.0.1:8082/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Say hello in one word"}], "max_tokens": 10}' \
  | jq '.choices[0].message.content'

# Qwen2.5-1.5B (:8084)
curl -s http://127.0.0.1:8084/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Say hello in one word"}], "max_tokens": 10}' \
  | jq '.choices[0].message.content'

# ── Test Agent GPU model ──
# Qwen3-235B (:8090) — this one takes a few seconds
curl -s http://127.0.0.1:8090/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Write a Java function that adds two integers. Keep it short."}],
    "max_tokens": 200,
    "temperature": 0.6
  }' | jq '.choices[0].message.content'
```

-----

## 11. Startup & Shutdown Procedures

### 11.1 Full Platform Start

```bash
# 1. Start all model containers
podman-compose -f /opt/ome-platform/podman-compose.yml up -d

# 2. Wait for all models to be ready
/opt/ome-platform/scripts/wait-for-models.sh 8081 8082 8083 8084 8085 8090

# 3. Start OME RAG Node.js
systemctl start ome-rag

# 4. Start OME Agent Node.js
systemctl start ome-agent

# 5. Verify
/opt/ome-platform/scripts/health-check.sh
```

### 11.2 Full Platform Stop

```bash
# Reverse order
systemctl stop ome-agent
systemctl stop ome-rag
podman-compose -f /opt/ome-platform/podman-compose.yml down
```

### 11.3 Restart Single Model (No Downtime for Others)

```bash
# Example: restart only the reranker
podman restart ome-bge-reranker

# Example: restart only the GPU model (takes 2-3 min)
podman restart ome-qwen3-235b
# Wait for it to be ready
/opt/ome-platform/scripts/wait-for-models.sh 8090
```

-----

## 12. Troubleshooting

### GPU Not Visible in Container

```bash
# Check CDI spec exists
ls -la /etc/cdi/nvidia.yaml

# Regenerate if missing
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# SELinux may block GPU access
sudo setsebool -P container_use_devices on

# Test GPU access
podman run --rm --device nvidia.com/gpu=all \
  nvidia/cuda:12.6.0-base-ubi9 nvidia-smi
```

### Qwen3-235B OOM (Out of Memory)

```bash
# Reduce context size
--ctx-size 32768   # Down from 65536

# Reduce parallel slots
--parallel 1       # Down from 2

# More aggressive MoE offloading (offload gate+up+down, not just experts)
-ot ".ffn_(gate|up|down)_exps.=CPU"
```

### SELinux Blocks Volume Mount

```bash
# Option 1: Use :Z flag (relabels)
-v /models/qwen3-235b-2507:/models:Z

# Option 2: Set SELinux context
sudo chcon -Rt svirt_sandbox_file_t /models/

# Option 3: Temporarily disable (not recommended for production)
sudo setenforce 0
```

### Split GGUF Files Not Auto-Loading

```bash
# All 3 files must be in the SAME directory
# Point --model to the FIRST file (00001-of-00003)
# llama.cpp automatically finds 00002 and 00003

# Verify:
ls /models/qwen3-235b-2507/UD-Q4_K_XL/
# Must show all 3 files
```

### Container Exits Immediately

```bash
# Check logs
podman logs ome-qwen3-235b

# Common causes:
# 1. Model file not found → check volume mount path
# 2. CUDA not available → check nvidia-ctk setup
# 3. Not enough RAM → check --memory flag
# 4. llama.cpp version too old → pull latest image
```

### Slow First Response (2-3 Minutes)

This is **normal** for the 235B model. The first request triggers loading 134 GB from disk into GPU VRAM + CPU RAM. Subsequent requests are 15-25 tok/s. The `--health-start-period=180s` accounts for this.

-----

## 13. Resource Allocation Summary

```
┌───────────────────────────────────────────────────────────────────┐
│  CPU Cores (128 total)                                             │
│                                                                    │
│  :8081 Arctic Embed         32 threads                             │
│  :8082 Qwen2.5-7B           16 threads                             │
│  :8083 BGE Reranker          8 threads                             │
│  :8084 Qwen2.5-1.5B          8 threads                             │
│  :8085 nomic-embed-code     24 threads                             │
│  :8090 Qwen3-235B MoE       32 threads (for expert computation)    │
│  ────────────────────────────                                      │
│  Subtotal:                  120 threads                             │
│  Remaining for OS/DB/Node:    8 cores                              │
│                                                                    │
│  Note: threads ≠ exclusive cores.                                  │
│  Models are not all active simultaneously.                         │
│  Actual concurrent CPU load is much lower.                         │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  RAM (1 TB total)                                                  │
│                                                                    │
│  Qwen3-235B MoE experts:     ~85 GB                                │
│  5 CPU model processes:      ~15 GB                                │
│  PostgreSQL:                 ~30 GB                                │
│  OME RAG Node.js:             ~4 GB                                │
│  OME Agent Node.js:           ~2 GB                                │
│  OS + filesystem cache:      ~50 GB                                │
│  ────────────────────────────                                      │
│  Used:                       ~186 GB                               │
│  Free:                       ~814 GB                               │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  GPU VRAM (93.6 GB)                                                │
│                                                                    │
│  Qwen3-235B non-MoE layers:  ~35 GB                               │
│  KV cache (64K × 2 slots):   ~50 GB                               │
│  ────────────────────────────                                      │
│  Used:                        ~85 GB                               │
│  Headroom:                     ~9 GB                               │
│                                                                    │
│  CPU models use ZERO GPU.                                          │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  Disk                                                              │
│                                                                    │
│  Qwen3-235B GGUF files:     134.3 GB                               │
│  5 CPU model GGUF files:      ~8.5 GB                              │
│  REPOS_DIR (47 repos):       ~10-50 GB (varies)                    │
│  PostgreSQL (otak schema):   ~20-50 GB (varies)                    │
│  ────────────────────────────                                      │
│  Total model storage:        ~143 GB                               │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  Network Ports                                                     │
│                                                                    │
│  :3000    OME RAG — MCP HTTP server                                │
│  :3100    OME Agent — MCP HTTP server                              │
│  :5432    PostgreSQL                                               │
│  :8081    Arctic Embed (prose embedding)                            │
│  :8082    Qwen2.5-7B (LLM generator)                               │
│  :8083    BGE Reranker (cross-encoder)                              │
│  :8084    Qwen2.5-1.5B (fast LLM)                                  │
│  :8085    nomic-embed-code (code embedding)                        │
│  :8090    Qwen3-235B (agent code generation)                       │
│                                                                    │
│  All on 127.0.0.1 (localhost only, not exposed externally)         │
│  except OME RAG/Agent HTTP which may be on 0.0.0.0                 │
└───────────────────────────────────────────────────────────────────┘
```
