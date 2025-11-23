# Setup Notes & Lessons Learned

Real-world implementation notes from setting up local codebase indexing with Ollama, Qwen3, Qdrant, and KiloCode.

---

## Key Discoveries

### 1. Ollama Outputs 4096 Dimensions by Default

**What we expected:** Based on research, 1024 dimensions seemed optimal (98-99% quality, 4× less storage)

**What actually happened:** Ollama's Qwen3-Embedding-8B-FP16 outputs **4096 dimensions** by default

**Verification:**
```bash
curl -s http://localhost:11434/api/embeddings -d '{
  "model": "qwen3-embedding:8b-fp16",
  "prompt": "test code snippet"
}' | jq '.embedding | length'
# Output: 4096
```

**Impact:**
- ✅ **Maximum quality** (100% model performance)
- ✅ **No configuration needed** (works out of the box)
- ⚠️ **4× storage** (~160MB vs ~40MB for 10K blocks)
- ⚠️ **Slightly slower search** (~20-30ms vs ~15ms)

**Decision:** Use 4096 dimensions for simplicity and maximum quality. With RTX 4090 and local storage, the trade-offs are acceptable.

---

### 2. KiloCode Auto-Creates Collections

**What we expected:** Need to manually create Qdrant collection before indexing

**What actually happened:** KiloCode automatically creates collections when you start indexing

**Collection naming:**
- Auto-generated name: `ws-{workspace-id}` (e.g., `ws-2e031c6c5c75628a`)
- Based on workspace directory
- One collection per workspace

**Settings applied:**
- Vector size: Matches configured model dimension (4096)
- Distance metric: Cosine
- On-disk storage: Enabled by default

**Benefit:** Zero manual setup - just configure KiloCode and click "Start Indexing"

---

### 3. Dimension Mismatch = "Bad Request" Error

**Error encountered:**
```
Status: Error - Failed during initial scan: Indexing failed:
Failed to process batch after 3 attempts: Bad Request
```

**Root cause:**
- KiloCode configured for 1024 dimensions
- Ollama outputting 4096 dimensions
- Qdrant collection created for 1024 dimensions
- Insertion failed: vector size mismatch

**Solution:**
1. Delete incorrect collection (via Qdrant dashboard)
2. Update KiloCode setting: Model dimension = 4096
3. Restart indexing (collection recreated correctly)

**Lesson:** Always verify actual embedding output dimensions before configuring

---

### 4. Docker Compose Healthchecks

**Initial docker-compose.yml had:**
```yaml
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:6333/healthz"]
```

**Problem:** Qdrant container doesn't include `wget` or `curl`

**Error:**
```
OCI runtime exec failed: exec failed: unable to start container process:
exec: "wget": executable file not found in $PATH
```

**Solution:** Remove healthcheck entirely (Qdrant official compose doesn't use one)

**Lesson:** Always check official Docker Compose examples before adding custom healthchecks

---

### 5. Optimal Search Configuration

**Default settings:**
- Max search results: 50
- Search score threshold: 0.40

**Testing results:**
- **Top result score:** 0.631 (good relevance)
- **50 results:** Provided comprehensive context
- **Documentation search:** Even 20-25 results sufficient
- **Code search:** 50 results recommended (distributed logic)

**Recommendation:**
- **Code work:** Keep 50 results (comprehensive understanding)
- **Simple docs:** 20-25 results sufficient
- **Threshold:** 0.40 works well (filters noise)

---

## What Works Great

### RAG for Code (Retrieval Augmented Generation)

**Complete workflow:**
1. User asks: "What are the hardware requirements?"
2. Chat model decides to use `codebase_search` tool
3. Query → Qwen3-Embedding → 4096-dim vector
4. Qdrant similarity search → Top 50 results
5. Results → Chat model context
6. Model generates answer from retrieved snippets

**Benefits observed:**
- ✅ No manual file pointing needed
- ✅ Finds relevant content across multiple files
- ✅ ~100ms search latency (very fast)
- ✅ Natural language queries work perfectly
- ✅ Semantic understanding (concepts, not just keywords)

**Example:**
- Query: "hardware requirements"
- Found: RTX 4090, VRAM specs, storage needs across 4 different markdown files
- Score: 0.631 relevance on top result

---

## Performance Metrics (RTX 4090)

### Indexing
- **This repo** (5 markdown files, ~800 blocks): ~2-3 minutes
- **GPU utilization:** Spikes to 100% during embedding generation
- **VRAM usage:** ~15GB (Qwen3 model)
- **Speed:** Very fast (RTX 4090 handles it easily)

### Search
- **Embedding generation:** 50-100ms
- **Vector search:** 10-20ms
- **Total latency:** ~100ms
- **Accuracy:** 0.631 top score (excellent)

### Storage
- **Qdrant collection:** ~3-5MB for this small repo
- **Model:** 15GB (one-time)
- **Docker volumes:** Minimal overhead

---

## Configuration Summary

### Final Working Configuration

**Ollama:**
- Model: `qwen3-embedding:8b-fp16`
- Base URL: `http://localhost:11434/`
- Output: 4096 dimensions (default)

**Qdrant:**
- URL: `http://localhost:6333`
- No API key (local deployment)
- Auto-created collection
- Distance: Cosine
- On-disk storage: Enabled

**KiloCode:**
- Provider: Ollama
- Model dimension: 4096
- Max results: 50
- Score threshold: 0.40
- Auto-create collection: Yes

**Docker:**
- Network: `ollama-network` (existing)
- No healthcheck (simplified)
- Persistent volume: `qdrant_storage`

---

## Troubleshooting Tips

### If indexing fails with "Bad Request"
1. Check actual embedding dimensions:
   ```bash
   curl -s http://localhost:11434/api/embeddings -d '{
     "model": "qwen3-embedding:8b-fp16",
     "prompt": "test"
   }' | jq '.embedding | length'
   ```
2. Verify KiloCode "Model dimension" matches output
3. Delete collection and let KiloCode recreate it

### If container won't start
1. Check Docker logs: `docker logs qdrant-kilocode`
2. Remove healthcheck if present
3. Verify `ollama-network` exists: `docker network ls`

### If search returns no results
1. Verify indexing completed (Green status)
2. Check collection has points: `curl http://localhost:6333/collections`
3. Lower score threshold below 0.40 if needed

---

## Recommended Workflow

### Setting Up New Project

1. **Pull model** (one-time):
   ```bash
   ollama pull qwen3-embedding:8b-fp16
   ```

2. **Deploy Qdrant** (if not running):
   ```bash
   docker compose up -d
   ```

3. **Configure KiloCode:**
   - Open project in VS Code
   - Settings → KiloCode → Codebase Indexing
   - Set model dimension to 4096
   - Save and Start Indexing

4. **Wait for indexing** (~10-20 min for typical project)

5. **Start searching** with natural language!

### Daily Usage

- Open project → KiloCode auto-detects existing index
- Code changes → Incremental updates automatically
- Switch projects → Each has its own collection
- No maintenance needed

---

## What We'd Change Next Time

### If Starting Fresh

1. **Skip manual collection creation** - KiloCode handles it
2. **Start with 4096 dimensions** - No need to configure Matryoshka
3. **Trust the defaults** - 50 results, 0.40 threshold work well
4. **Use official Docker Compose** - No custom healthchecks

### Future Optimizations

**If storage becomes an issue:**
- Configure Matryoshka for 1024 dimensions (via Ollama modelfile)
- Trade 1% quality for 4× storage savings

**If search is too slow:**
- Reduce max results to 20-25
- Increase score threshold to 0.50
- Enable quantization in Qdrant

**For very large codebases:**
- Consider Qdrant clustering
- Enable payload indexing for faster filtering
- Use on-disk payload storage

---

## Architecture Validation

### What Works As Expected

✅ **Semantic search** - Understands concepts, not just keywords
✅ **RAG workflow** - Retrieval → Augmentation → Generation
✅ **Local privacy** - No data leaves machine
✅ **Cost efficiency** - No API fees, unlimited queries
✅ **Performance** - RTX 4090 handles it effortlessly

### What Surprised Us

😮 **4096 dimensions default** - Expected 1024 from research
😮 **Auto-collection creation** - Thought manual setup required
😮 **Indexing speed** - Faster than expected (~2-3 min for this repo)
😮 **Search quality** - 0.631 relevance scores excellent

---

## Conclusion

This setup provides **production-grade RAG for code** with:

- State-of-the-art embeddings (Qwen3 #1 MTEB ranked)
- Local deployment (100% privacy)
- Minimal configuration (KiloCode auto-creates)
- Excellent performance (RTX 4090 optimized)
- Zero ongoing costs

**Total setup time:** ~30 minutes (including troubleshooting)
**Result:** Semantic code search across entire codebase with natural language

**Success!** 🎉
