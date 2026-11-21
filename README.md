# Bonsai 2 27B — Optimized Docker Setup for 12 GB VRAM

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

### 4\. Create volume


```
docker volume create local-llama-bonsai2_llama_models
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

## 🧠 Agent Harness Compatibility

* ✅ Validated in multi-hour agentic sessions with  **Goose** — MCP extensions, long agent loops, context compression, skill library
* ✅ **Any OpenAI-compatible client** (`/v1/chat/completions`), but first remove the five Goose-only keys below

**Important:** your agent harness must trigger context compression before the window fills. Tested thresholds: 75% of `CTX\\\_SIZE` (49152 tokens at 65,536).

### Goose Compatibility

#### Prompts 

Copy the overrides from this repository into the Goose config directory (Windows example):
```
"%APPDATA%\Block\goose\config\prompts"
```

Or paste them in Goose Desktop → Settings → Prompts → Edit (system.md, compaction.md). Enable the Todo extension under Settings → Extensions. Start a new session — templates are read at session start and override the built-ins permanently.

**system.md** — what changed and why (required Goose Desktop Windows):

Windows client only: the prompts assume cmd.exe. 

| Change | Why |
| --- | --- |
| MOIM passthrough block (`moim_system_prompt_block`) | Without it the todo extension stores notes the model never sees again |
| cmd.exe command cookbook, `&&` for dependent steps, `findstr` fallback | The default prompt gives no shell guidance; broken placeholders and PowerShell syntax waste turns |
| Working state protocol (`plan.md`) | Durable task state; after compaction the agent resumes instead of re-exploring the project |
| Safety rules, no subagent delegation | PID-only kills (a broad `taskkill` kills Goose's own extension servers); one inference slot |
| Static rules first, dynamic blocks last | Keeps the cached prompt prefix stable |

**compaction.md** — structured JSON summary that keeps file paths, commands and unresolved errors verbatim, the checklist state, and the location of plan.md with the next action. Stock summarisation drops exactly the details needed to continue.

#### Tools

**CHAT_KWARGS**

Goose exposes tools as extension__tool, and the template matches names exactly — upstream defaults do not recognise Goose, so several agent layers stay silently disabled. The shipped .env sets:

| kwarg | Value | Why |
| --- | --- | --- |
| `todo_tools` | `todo_write todo__todo_write …` | Goose's actual tool names; otherwise todo-aware layers never fire |
| `todo_nudge_after` | `0` | Goose's todo extension forbids re-verifying completed items; template reconciliation would contradict it. Durability comes from `plan.md` + MOIM |
| `mutating_tools` | `write edit developer__write …` | Goose write tools exempt from loop detection. `shell` is deliberately excluded — repeated identical shell output is the real stuck signal |
| `read_tools` | `tree developer__tree …` | Read tools exempt from content-based failure markers |
| `warn_vanished_tools` | `false` | Goose toggles extensions per request; the "tool server died" note would be a false positive |
| `enable_skill_routing` | `false` | No skill catalog here; removes routing instructions from every prompt |
| `skill_loader_tools` | `load summon__load …` | Goose ≥ 1.25 loads skills via Summon; kept for future use |
| `max_tool_arg_chars` | `8000` | Raised from 3000 — over-truncated write arguments force re-reads of files just written |
| `max_tool_response_chars` | `6000` | Bounds the cost of a single command |
| `long_run_nudge_after` | `32` | Earlier closeout checkpoint, sized for single-slot throughput |
| `tool_call_format` / `reasoning_effort` | `xml` / `medium` | Pinned: XML keeps grammar enforcement, `medium` avoids reasoning blowup and truncation |

Three rules when editing: keep everything in one JSON object (a second flag overrides, not merges); set kwargs on the server command line (some harnesses drop client-side values); mutating_tools and read_tools replace the upstream lists — extend them when adding MCP extensions.

#### Goose-only kwargs

These five keys are tuned for Goose and **degrade other harnesses**. Remove them from `CHAT_KWARGS` to fall back to the template's cross-harness defaults.

| kwarg | Value here | Effect on other harnesses |
| --- | --- | --- |
| `mutating_tools` | `write edit developer__write developer__edit todo_write todo__todo_write` | **Replaces** the upstream ~55-name union. Other harnesses' write tools lose their no-progress exemption → false loop warnings on productive writes |
| `read_tools` | `tree developer__tree read read_file view` | **Replaces** the upstream ~30-name union. Other harnesses' read tools lose the failure-marker exemption → error-shaped file content misread as tool failure |
| `warn_vanished_tools` | `false` | Silences the dead-MCP-server note. Needed for Goose (it toggles extensions per request), but harnesses with a static tool list lose a real diagnostic |
| `enable_skill_routing` | `false` | Turns off skill routing. Harnesses that publish a skill catalog (e.g. OpenCode) stop loading skills |
| `todo_nudge_after` | `0` | Disables plan reconciliation. Matches Goose's lightweight todo notes, but disables useful checkpoints elsewhere |

Everything else in `CHAT_KWARGS` is portable: `todo_tools` and `skill_loader_tools` keep the upstream names and only add Goose's, so they are inert elsewhere; the remaining keys are harness-neutral.

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
