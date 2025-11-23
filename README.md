# Local Codebase Indexing for KiloCode

> State-of-the-art semantic code search powered by local AI - optimized for RTX 4090

A complete local setup for intelligent codebase indexing using Ollama, Qwen3 embeddings, Qdrant vector database, and the KiloCode VS Code extension. Provides natural language code search with local embeddings, no API costs, and no rate limits. (Full privacy requires both local embeddings AND local inference LLM.)

---

## Table of Contents

### Overview
- [About This Project](#about-this-project)
- [What This Is (RAG for Code)](#what-this-is-rag-for-code)
  - [The Problem → Solution](#the-problem--solution)
  - [Why It Matters](#why-it-matters)
  - [How RAG Works](#how-rag-works-conceptual-overview)
  - [Why Local?](#why-local)

### Technical Details
- [Tech Stack](#tech-stack)
- [Key Features](#key-features)
- [Why Qwen3](#why-qwen3-embedding-8b)
- [Why 4096 Dimensions](#why-4096-dimensions)
- [Architecture](#architecture)
- [Performance Expectations](#performance-expectations)
- [Cost Analysis](#cost-analysis)

### Getting Started
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [My Docker Setup](#my-docker-setup-reference-only)

### Reference
- [Documentation](#documentation)
- [Configuration Details](#configuration-details)
- [Troubleshooting](#troubleshooting)
- [Project Status](#project-status)
- [Technical References](#technical-references)
- [License](#license)
- [Contributing](#contributing)
- [Acknowledgments](#acknowledgments)

---

## About This Project

This repository documents setting up **local RAG (Retrieval Augmented Generation)** for code using [KiloCode](https://kilo.ai/) - my **preferred AI code assistant as of November 23rd, 2025** for its user-friendly interface and comprehensive features.

**Purpose:** Make codebase indexing accessible through clear, user-friendly documentation. This is technical research simplified for developers who want to understand WHY and HOW RAG works, not just copy/paste commands.

**What's included:**
- Plain-English explanations of embeddings, vectors, and semantic search
- Research on embedding models and why Qwen3-8B was chosen
- Hardware-optimized setup guide (RTX 4090 + [Ollama](https://ollama.com/) + [Qdrant](https://qdrant.tech/))
- **Complete Qdrant installation via Docker Compose**
- Full documentation on architecture, performance, and troubleshooting

**Prerequisites (not covered here):**
- [Docker](https://www.docker.com/) - Already installed and configured
- [Ollama](https://ollama.com/) - Already running with models pulled
- Basic familiarity with terminal/command line

**Starting point:** Based on [KiloCode's Codebase Indexing Documentation](https://kilo.ai/docs/features/codebase-indexing), customized for my hardware and needs.

**Note:** The `docker-compose.yml` is tailored to my setup (e.g., `ollama-network`), but easily adaptable as a template. While focused on KiloCode, the RAG principles apply to any AI code assistant (Cursor, Continue, Aider, etc.).

## What This Is (RAG for Code)

This is **RAG (Retrieval Augmented Generation)** - giving AI models accurate context from YOUR codebase before they answer questions.

### The Problem → Solution

| Without RAG | With RAG (This Project) |
|-------------|-------------------------|
| **Q:** "What database do we use?" | **Q:** "What database do we use?" |
| **A:** "After manually reading package.json, database config files, and connection modules... PostgreSQL 14, configured in `config/database.js:12`" (correct, but inefficient - read 5+ files, wasted tokens) | **A:** "PostgreSQL 14, configured in `config/database.js:12` with connection pooling (max 20 connections)" (accurate, instant) |
| | |
| **Q:** "How do I add authentication?" | **Q:** "How do I add authentication?" |
| **A:** "After trial and error reading auth.ts, jwt.service.ts, middleware files... your project uses JWT with refresh tokens in `auth/jwt.service.ts:34-67`" (correct, but slow - read 10+ files manually) | **A:** "Your project uses JWT with refresh tokens. See `auth/jwt.service.ts:34-67` for token generation and `middleware/auth.ts:12-28` for verification" (specific, instant) |

**The key difference:** RAG's semantic search finds code by meaning (not just keywords), enabling natural language queries across different naming conventions. Without RAG, you need exact terminology or better prompting to identify what to search for.

**Analogy:** Like Google's semantic search vs. Ctrl+F/grep keyword matching - both find things fast, but semantic search understands meaning (finds "authentication" when you search "user verification"), while keyword search requires knowing the exact terms used in the code.

---

### Why It Matters

✅ **Accuracy** - Answers based on YOUR code, not assumptions
✅ **Efficiency** - Instant semantic search, no manual file hunting
✅ **Reduced Hallucination** - Grounded in actual code
✅ **Context-Aware** - Understands your architecture and patterns
✅ **Privacy** - Local embeddings + storage (full privacy requires local inference LLM)
✅ **Cost** - No API fees for local embeddings + storage (cloud options available with associated costs)
✅ **Works with smaller models** - Good context = great answers

⚠️ **Trade-offs:** ~30min setup, GPU required (15GB VRAM), storage (~160MB per 10K blocks)

**Worth it?** If you work with large codebases and want accurate AI assistance, absolutely yes.

---

### How RAG Works (Conceptual Overview)

RAG works in three simple steps:

1. **Index Once** - Parse your codebase, convert semantic blocks (functions, classes) into vector embeddings, store in Qdrant
2. **Auto-Update** - File watcher detects changes, automatically re-indexes modified files only (fast, incremental)
3. **Search Instantly** - When you ask a question, convert query to vector, find similar code via semantic search, inject into LLM context

**The magic:** Instead of reading files randomly, the system performs semantic search to find ONLY the relevant code snippets. Your question "how does auth work?" instantly retrieves authentication-related functions across the entire codebase, then feeds them to the LLM for accurate answers.

**This is production-grade RAG** - the same technology behind ChatGPT's "custom GPTs" and enterprise AI assistants, but 100% local.

**For detailed technical flows with KiloCode orchestration:** See [Architecture](#architecture) section below (Initial Indexing, Auto-Update, and Search flows).

---

### Why Local?

- **Privacy:** Embeddings generated AND stored locally in Qdrant (full privacy requires local inference LLM for KiloCode responses)
- **Cost:** Minimal ongoing electricity costs vs per-usage cloud API fees
- **Performance:** No network latency, no rate limits
- **Control:** Your hardware, your rules
- **No vendor lock-in:** Works offline, independent of cloud services

**Note:** This setup uses local embedding generation + local vector storage. You could optionally use cloud-based Qdrant or cloud embedding providers if your privacy requirements differ.

## Tech Stack

| Component | Technology | Specification |
|-----------|-----------|---------------|
| **Embedding Model** | Qwen3-Embedding-8B-FP16 | #1 MTEB ranked (80.68 code, 70.58 ML) |
| **Dimensions** | 4096 | 100% quality, Qwen3-8B via Ollama output |
| **Vector Database** | Qdrant | Cosine similarity, local deployment |
| **AI Runtime** | Ollama | Docker-based, GPU accelerated |
| **Code Parser** | Tree-sitter | AST-based semantic blocks |
| **Interface** | KiloCode | VS Code extension |
| **Hardware** | RTX 4090 | 15GB VRAM required |

## Key Features

### Performance
- **Search Latency:** Fast local search (milliseconds)
- **Indexing Speed:** Varies by codebase size (minutes to hours for initial indexing)
- **Accuracy:** High top-10 retrieval accuracy
- **Context Window:** 32K tokens (handle large functions/files)

### Capabilities
- Natural language queries ("authentication logic", "error handling patterns")
- Semantic understanding across 100+ programming languages
- Incremental indexing with file watching
- Git branch change detection
- Automatic file filtering (binaries, dependencies, large files)
- Hash-based caching for efficiency

### Resource Usage
- **Model VRAM:** ~15GB FP16 (maximum quality)
- **Qdrant RAM:** ~60-100MB for typical codebase
- **Storage:** ~160MB vectors for 10K code blocks (4096 dims)
- **Throughput:** GPU-accelerated embedding generation

## Why Qwen3-Embedding-8B?

After evaluating 9 embedding models, Qwen3-Embedding-8B was selected for:

1. **Best-in-class performance** - #1 on MTEB Code benchmark
2. **Code-optimized training** - 100+ programming languages
3. **Hardware compatibility** - 15GB fits in RTX 4090's 24GB VRAM
4. **Advanced features** - Instruction-aware, Matryoshka support, 32K context
5. **Future-proof** - Latest release (June 2025), Apache 2.0 license

See [2_EMBEDDING_MODEL_SELECTION.md](2_EMBEDDING_MODEL_SELECTION.md) for detailed comparison.

## Why 4096 Dimensions?

Qwen3 supports Matryoshka embeddings (32-4096 dimensions), but **Qwen3-8B via Ollama outputs 4096**:

- **Quality:** 100% maximum performance (no quality loss)
- **Simplicity:** No configuration needed, works out of the box
- **Speed:** Fast search with GPU acceleration
- **Storage:** ~160MB for 10K blocks (acceptable with local setup)

**Note:** While 1024 dimensions would be more storage-efficient (98-99% quality), using the model's 4096 output as-is eliminates configuration complexity and provides maximum quality for local deployments.

| Dimensions | Quality | Use Case |
|------------|---------|----------|
| 256 | ~95% | Mobile/massive scale |
| 512 | ~97% | Large scale deployments |
| 1024 | ~99% | Balanced quality/storage |
| 2048 | ~99.7% | Specialized domains |
| **4096** | **100%** | **This project (model default)** |

**Note:** While the model supports Matryoshka embeddings (configurable dimensions), when using Qwen3-Embedding-8B-FP16 through Ollama it outputs **4096 dimensions**. This provides maximum quality with no configuration needed.

## Prerequisites

### Hardware
- **GPU:** NVIDIA GPU with **16GB+ VRAM** (Qwen3-Embedding-8B-FP16 requires ~15GB)
  - **This project uses:** RTX 4090 (24GB) for maximum performance
  - **Alternatives:** Smaller embedding models available for lower VRAM cards (see research docs)
  - **Budget option:** Quantized models reduce VRAM further (with minor quality trade-off)
  - **Note:** This setup targets state-of-the-art local performance; adjust model choice for your hardware
- **RAM:** 16GB+ system RAM recommended
- **Storage:** 50GB+ free space (model + vectors + Docker)

### Software
- **Docker & Docker Compose** - Container orchestration
- **Ollama** - Already installed and running
- **KiloCode Extension** - VS Code extension for codebase indexing
- **Existing ollama-network** - Docker network (already configured)

### Verification
```bash
# Check Ollama is running
docker ps | grep ollama

# Verify ollama-network exists
docker network ls | grep ollama-network

# Confirm GPU is available
nvidia-smi
```

## Quick Start

### 1. Pull the Embedding Model
```bash
ollama pull qwen3-embedding:8b-fp16
```

### 2. Deploy Qdrant
```bash
docker compose up -d
```

Verify it's running:
```bash
docker ps | grep qdrant
curl http://localhost:6333/healthz
```

Dashboard available at: http://localhost:6333/dashboard

### 3. Configure KiloCode
1. Open VS Code with KiloCode extension
2. Navigate to Settings → KiloCode → Codebase Indexing
3. Configure:
   - **Enable:** ✓ Codebase Indexing
   - **Provider:** Ollama
   - **Ollama base URL:** http://localhost:11434/
   - **Model:** qwen3-embedding:8b-fp16
   - **Model dimension:** 4096
   - **Qdrant URL:** http://localhost:6333
   - **Qdrant API key:** (leave empty for local)
   - **Max Results:** 50 (adjustable based on your needs)
   - **Search score threshold:** 0.40 (default)

### 4. Start Indexing
1. Click **Save** in KiloCode settings
2. Click **Start Indexing**
3. KiloCode will:
   - **Auto-create collection** (workspace-based name)
   - Parse code with Tree-sitter
   - Generate embeddings via Ollama
   - Store vectors in Qdrant
4. Watch status: Gray → Yellow (indexing) → Green (ready)
5. Indexing time: Varies by project size (GPU-accelerated)

### 5. Search!
```
Natural language queries in KiloCode:
- "user authentication logic"
- "error handling patterns"
- "database connection setup"
- "API endpoint definitions"
```

---

## My Docker Setup (Reference Only)

**Important:** This documentation is optimized for MY specific environment but designed as a reference for others to adapt.

For context, here's how I run Ollama in Docker:

```bash
docker run -d \
  --network ollama-network \
  --gpus device=all \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  -e OLLAMA_FLASH_ATTENTION=1 \
  -e OLLAMA_KV_CACHE_TYPE=q8_0 \
  ollama/ollama
```

**Key parameters explained:**
- `--network ollama-network`: **OPTIONAL** - My custom Docker network. You can omit this, use your own network, or run without one entirely.
- `OLLAMA_FLASH_ATTENTION=1`: **Optional (recommended)** - Performance optimization for faster inference.
- `OLLAMA_KV_CACHE_TYPE=q8_0`: **Optional** - Enables lower VRAM usage at minimal accuracy cost.

**Adapt to your needs:** Don't blindly copy the network setting. Either remove it, use your existing Docker network, or create your own. The setup works perfectly fine without custom networks.

For detailed Ollama configuration options, see the [official Ollama documentation](https://github.com/ollama/ollama/blob/main/docs/docker.md).

---

## Documentation

This repository contains comprehensive documentation organized by workflow:

| Document | Description |
|----------|-------------|
| [1_CODEBASE_INDEXING_FEATURE.md](1_CODEBASE_INDEXING_FEATURE.md) | **Codebase Indexing Feature** - Overview of the codebase indexing feature from official KiloCode documentation |
| [2_EMBEDDING_MODEL_SELECTION.md](2_EMBEDDING_MODEL_SELECTION.md) | **Embedding Model Selection** - Research and comparison of 9 embedding models, why Qwen3-8B was chosen |
| [3_QWEN3_OLLAMA_GUIDE.md](3_QWEN3_OLLAMA_GUIDE.md) | **Qwen3 with Ollama Guide** - Why the default configuration is perfect, FAQ, and best practices |
| [4_QDRANT_INSTALLATION_GUIDE.md](4_QDRANT_INSTALLATION_GUIDE.md) | **Qdrant Installation** - Step-by-step Docker Compose deployment, configuration, integration with ollama-network |
| [FAQ.md](FAQ.md) | **Frequently Asked Questions** - Quick reference for understanding RAG, collections, vectors, and workflow |
| [IMPLEMENTATION_NOTES.md](IMPLEMENTATION_NOTES.md) | **Lessons Learned** - Real-world implementation notes, key discoveries, troubleshooting tips |

## Architecture

### Initial Indexing Flow (First Time Setup)
```
User clicks "Start Indexing" in KiloCode
  ↓
KiloCode checks if Qdrant collection exists
  ↓ No collection found
KiloCode creates new collection (workspace-based name)
  ↓
KiloCode scans entire codebase
  ↓ Reads all code files
KiloCode parses files with Tree-sitter
  ↓ Extracts all semantic blocks
Semantic Blocks (functions, classes, methods)
  ↓ KiloCode sends blocks to Ollama API (batched)
Ollama + Qwen3-Embedding-8B-FP16
  ↓ Creates 4096-dim vector embeddings
  ↓ Returns vectors to KiloCode
KiloCode sends vectors + metadata to Qdrant
  ↓ Stores all vectors
Qdrant Collection (kilocode_codebase)
  ↓ Persistent storage
Indexed Codebase ✅
  ↓
Status: Gray → Yellow (indexing) → Green (ready)
```

### Auto-Update Flow (File Changes)
```
File watcher detects change (save/delete/create)
  ↓
KiloCode identifies changed file(s)
  ↓
KiloCode deletes old vectors for that file from Qdrant
  ↓
KiloCode re-parses changed file with Tree-sitter
  ↓ Extracts new/modified blocks
Semantic Blocks (updated)
  ↓ KiloCode sends blocks to Ollama API
Ollama + Qwen3-Embedding-8B-FP16
  ↓ Creates 4096-dim vector embeddings
  ↓ Returns vectors to KiloCode
KiloCode sends updated vectors + metadata to Qdrant
  ↓ Replaces old vectors
Qdrant Collection (kilocode_codebase)
  ↓ Updated storage
Index stays current ✅ (incremental, fast)
```

### Data Flow: Search
```
Natural Language Query
  ↓
KiloCode (VS Code Extension)
  ↓ Sends query to Ollama API
Ollama + Qwen3-Embedding-8B-FP16
  ↓ Creates 4096-dim vector embedding
  ↓ Returns vector to KiloCode
KiloCode sends vector to Qdrant
  ↓
Qdrant Vector Database
  ↓ Cosine similarity search
  ↓ Returns top-N matches
Code Snippets + Metadata (file, lines, score)
  ↓ Sent back to KiloCode
KiloCode Results Panel
```

### Network Architecture
```
Docker Network: ollama-network
├── Ollama Container (GPU accelerated)
│   └── qwen3-embedding:8b-fp16
└── Qdrant Container
    ├── HTTP API: localhost:6333
    ├── gRPC: localhost:6334
    └── Dashboard: localhost:6333/dashboard

KiloCode (host) → localhost:6333 → Qdrant
KiloCode (host) → localhost:11434 → Ollama
Qdrant → qdrant:11434 → Ollama (internal)
```

## Configuration Details

### Ollama Model Settings
```bash
# Model: qwen3-embedding:8b-fp16
# No custom modelfile needed - use default
# Context: 32K tokens (built-in)
# Output: 4096 dimensions (Qwen3-8B via Ollama)
# Quantization: FP16 for maximum quality
```

### Qdrant Collection Settings
```
Collection Name: kilocode_codebase
Vector Dimensions: 4096
Distance Metric: Cosine
Indexing: HNSW (Hierarchical Navigable Small World)
```

### KiloCode Settings
```
Embedding Provider: Ollama
Model: qwen3-embedding:8b-fp16
Qdrant URL: http://localhost:6333
Max Search Results: 50
Min Block Size: 100 chars
Max Block Size: 1000 chars
```

## Performance Expectations

### Indexing Performance
| Codebase Size | Blocks | VRAM | Storage |
|---------------|--------|------|---------|
| Small (1K files) | ~5K | 15GB | 20MB |
| Medium (5K files) | ~25K | 15GB | 100MB |
| Large (10K files) | ~50K | 15GB | 200MB |

**Indexing Time:** Varies by project size and hardware. GPU-accelerated embedding generation is the bottleneck. Initial indexing may take minutes to hours depending on codebase size.

### Search Performance
- **Search Latency:** Fast local search in milliseconds
- **Embedding Generation:** GPU-accelerated (Ollama)
- **Vector Search:** Fast similarity matching (Qdrant)
- **Accuracy:** High top-10 retrieval accuracy

### Resource Monitoring
```bash
# GPU utilization
nvidia-smi

# Qdrant dashboard
http://localhost:6333/dashboard

# Check collection stats
curl http://localhost:6333/collections/kilocode_codebase
```

## Cost Analysis

### Local Setup (This Project)
- **Hardware:** One-time (already owned RTX 4090)
- **Electricity:** Ongoing cost (GPU running intermittently, varies by local rates and usage)
- **Per-query cost:** Effectively $0 (unlimited)

### Cloud API Alternative
- **Pricing model:** Per-token/per-query charges
- **Usage costs:** Scale with usage (light to heavy)
- **Rate limits:** Applied
- **Privacy:** Code sent to cloud

**Winner:** Local setup - unlimited queries, local embeddings + storage (complete privacy requires local inference LLM), minimal ongoing costs vs per-usage cloud fees

## Project Status

### ✅ Completed
- Research and model selection (Qwen3-Embedding-8B)
- Architecture design
- Documentation (comprehensive guides)
- Docker Compose file for Qdrant
- Qdrant deployment and configuration
- KiloCode integration and testing
- Hardware compatibility verification
- Performance validation
- Implementation documentation

### 🔮 Future Enhancements
- Automatic backup/restore scripts
- Multi-workspace support
- Custom block sizing strategies
- Performance monitoring dashboard

## Troubleshooting

### Common Issues

**Q: Indexing is slow**
- Check GPU utilization: `nvidia-smi`
- Verify FP16 model: `ollama list | grep qwen3`
- Check Docker resources

**Q: Search returns poor results**
- Verify 4096 dimensions configured
- Check collection exists: `curl localhost:6333/collections`
- Rebuild index in KiloCode settings

**Q: High VRAM usage**
- Expected: ~15GB for FP16 model
- Consider smaller model if needed (qwen3:4b)
- Close other GPU applications

## Technical References

- **Qwen3 Model Card:** [Qwen/Qwen3-Embedding-8B](https://huggingface.co/Qwen/Qwen3-Embedding-8B)
- **MTEB Leaderboard:** [Code Embedding Benchmark](https://huggingface.co/spaces/mteb/leaderboard)
- **Qdrant Docs:** [https://qdrant.tech/documentation/](https://qdrant.tech/documentation/)
- **Ollama Docs:** [https://ollama.ai/docs](https://ollama.ai/docs)
- **KiloCode:** [VS Code Extension](https://marketplace.visualstudio.com/items?itemName=kilocode)

## License

This project configuration and documentation is provided as-is for personal use.

**Component Licenses:**
- Qwen3-Embedding-8B: Apache 2.0
- Qdrant: Apache 2.0
- Ollama: MIT
- KiloCode: Check extension license

## Contributing

This is a personal project documenting a local setup. Feel free to:
- Fork and adapt for your hardware
- Submit issues for documentation improvements
- Share your own optimizations

## Acknowledgments

- **Alibaba Cloud** - Qwen3 embedding models
- **Qdrant** - High-performance vector database
- **Ollama** - Simple local AI deployment
- **KiloCode** - Semantic code search integration

---

**Ready to get started?** Follow the [4_QDRANT_INSTALLATION_GUIDE.md](4_QDRANT_INSTALLATION_GUIDE.md) for step-by-step deployment instructions.
