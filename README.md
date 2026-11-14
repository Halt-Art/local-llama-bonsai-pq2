# Bonsai 2 27B — Optimized Docker Setup for RTX 3060

Production-ready Docker Compose setup for serving **Bonsai 2 27B** (ternary PQ2\_0 quantization) with the PrismML llama.cpp fork. Tuned for NVIDIA Ampere GPUs with 12 GB VRAM and validated in long-running agentic sessions (Goose, Hermes).

## 🚀 Features

* **Ternary PQ2\_0 quantization** — \~7.2 GB, 27B-class reasoning
* **65K token context window** with Q4\_0 KV-cache, fits in 12 GB VRAM
* **Agentic Jinja chat template** ([Moore2877/Qwen-Fixed-Chat-Templates-llamacpp](https://huggingface.co/Moore2877/Qwen-Fixed-Chat-Templates-llamacpp)) with XML tool calls, reasoning-effort steering, loop detection and skill/todo kwargs
* **Anti-repetition sampler stack**: presence, frequency and repetition penalties tuned for long agent loops
* **Automated bootstrap**: model GGUF and chat template are downloaded from Hugging Face into a shared Docker volume on first start
* **Ollama sidecar** with nomic-embed-text for embeddings (CPU-only, loaded into RAM with zero GPU overhead)

## 📋 Requirements

* **GPU**: NVIDIA GPU (Ampere or newer recommended)
* **VRAM**: 12GB minimum (tested on RTX 3060 12GB)
* **CUDA**: 12.4 container runtime (host driver ≥ 550.54 / CUDA 12.4+ compatible)

### Tested Configuration

|Component|Specification|
|-|-|
|GPU|NVIDIA RTX 3060 12GB|
|Driver|576.02|
|CUDA|12.9|
|OS|Windows 10|
|Inference Speed|\~25 t/s|

## 🧪 Agent Harness Compatibility

* ✅ Validated in multi-hour agentic sessions with  **Goose** — MCP extensions, long agent loops, context compression, skill library
* ✅ **Hermes** -  Verified in lighter usage
* ✅ **Any OpenAI-compatible client** (`/v1/chat/completions`)

**Important:** your agent harness must trigger context compression before the window fills. Tested thresholds: 75% of `CTX\\\_SIZE` (49152 tokens at 65,536).

## ⚡ Quick Start

### 1\. Clone the repository

```
git clone https://github.com/Halt-Art/local-llama-bonsai-pq2.git
cd local-llama-bonsai-pq2
```

### 2\. Build the Docker image

The image is built locally from the PrismML llama.cpp fork with custom ternary CUDA kernels:

```
docker compose -f docker-compose-pq2.yml --env-file .env build llama-server
```

### 3\. Bootstrap model and chat template

The init services download the GGUF model (\~7.2 GB) and the agentic Jinja template from Hugging Face into a shared Docker volume:

```
docker compose -f docker-compose-pq2.yml --env-file .env run --rm init-template
docker compose -f docker-compose-pq2.yml --env-file .env run --rm init-models
```

### 5\. Start the server

```
docker compose -f docker-compose-pq2.yml --env-file .env up -d llama-server
```

### 6\. Verify after launch

Check container states (`llama-server-pq2` healthy, init containers exited 0):

```
docker compose -f docker-compose-pq2.yml ps
```

API health-check:

```
curl http://localhost:8080/v1/models
curl http://localhost:11435/api/tags
```

Verify the flags the server actually received:

```
docker exec llama-server-pq2 sh -c "tr '\\\\0' '\\\\n' < /proc/1/cmdline"
```

Watch logs in real time:

```
docker logs -f llama-server-pq2
```

### 7\. An optional step to perform if you have changed the launch parameters

```bash
docker compose -f docker-compose-pq2.yml down
docker compose -f docker-compose-pq2.yml up -d
```

## 📖 Upstream Projects

* [Bonsai 2 27B (GGUF)](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) — ternary 27B reasoning model by Prism ML
* [PrismML llama.cpp fork](https://github.com/PrismML-Eng/llama.cpp) — custom ternary hybrid-attention CUDA kernels
* [Qwen Fixed Chat Templates](https://huggingface.co/Moore2877/Qwen-Fixed-Chat-Templates-llamacpp) — agentic Jinja template with loop detection and skill routing
* [Ollama](https://github.com/ollama/ollama) — local model runtime powering the embedding sidecar
* [nomic-embed-text](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5) — embedding model served by the Ollama sidecar for retrieval / RAG workloads

## 📄 License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

### Component Licenses

This project includes the following components under their respective licenses:

* [Bonsai 2 27B](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) by Prism ML - [Apache 2.0](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf#license)
* [llama.cpp fork](https://github.com/PrismML-Eng/llama.cpp) by PrismML-Eng - [MIT License](https://github.com/PrismML-Eng/llama.cpp/blob/main/LICENSE)
* [Qwen Chat Template](https://huggingface.co/Moore2877/Qwen-Fixed-Chat-Templates-llamacpp) by Moore2877 - [Apache 2.0](https://huggingface.co/Moore2877/Qwen-Fixed-Chat-Templates-llamacpp#credits-and-license)
* [Ollama](https://github.com/ollama/ollama) by Ollama - [MIT License](https://github.com/ollama/ollama/blob/main/LICENSE)
* [nomic-embed-text](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5) by Nomic AI - [Apache 2.0](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5)

