# Qdrant & KiloCode Integration FAQ

Quick reference guide for understanding Qdrant collections and KiloCode workflow.

---

## Understanding Collections

### What is a collection and why do we need it?

A **collection** is like a database table, but for vectors (embeddings). It stores:
- The 1024-number vectors from your code embeddings
- Metadata (file path, line numbers, code snippet text)

You need it because Qdrant needs a structured place to store and search your code embeddings efficiently.

---

### Will a collection be for all projects? Or one per project?

**Best practice: ONE collection per codebase/project**

Each project/repo should have its own collection:
- ✅ `my_webapp_codebase` - for your web app project
- ✅ `my_api_codebase` - for your API project
- ✅ `kilocode_codebase` - for testing/this project

**Why separate collections?**
- Keeps different projects isolated
- Prevents mixing unrelated code in search results
- Better performance (searching smaller, focused datasets)
- Cleaner organization

**NOT recommended:** One giant collection for all projects - that would return irrelevant results from unrelated codebases!

---

## Understanding Vectors

### What is vector dimension?

Think of it like **resolution** or **detail level** for your code embeddings.

- Each code snippet becomes a vector of numbers
- More dimensions = more detail = better accuracy
- **Qwen3 via Ollama outputs 4096 dimensions by default**
- Qdrant needs to know "expect 4096 numbers per vector" to store them correctly

**Analogy:** Like telling a photo app "all images will be 3840x2160" - Qdrant needs to know the vector "size" upfront.

**For this project:** Use **4096** (Ollama's default output for Qwen3-Embedding-8B)

**Note:** While Qwen3 supports Matryoshka embeddings (configurable 32-4096 dimensions), Ollama's implementation defaults to the full 4096 for maximum quality. This requires no configuration and provides 100% model performance.

---

### What is Cosine and why is it important?

**Distance metric** = "How do we measure similarity between two vectors?"

**Cosine** measures the *angle* between vectors:
- `1.0` = identical (0° angle, perfect match)
- `0.0` = completely different (90° angle, unrelated)

**Why Cosine for code search?**
- Focuses on *direction* (meaning/semantics) not magnitude
- Works great for embeddings from language models
- Industry standard for semantic search
- Captures conceptual similarity, not just word matching

**Example:** "authentication code" and "user login logic" point in similar directions (similar meaning) even though they use different words.

**For this project:** Always use **Cosine** distance metric

---

## KiloCode Workflow

### How does indexing work with multiple projects?

**Per-Project Workflow:**

1. **Create collection** (one-time per project)
   - Collection name: `my_project_name`
   - Vector size: 1024
   - Distance: Cosine

2. **Configure KiloCode** (one-time per project)
   - Point to the collection name
   - Set Qdrant URL: `http://localhost:6333`

3. **Open project in VS Code**
   - KiloCode automatically starts indexing

4. **Work normally**
   - KiloCode monitors file changes
   - Incremental updates happen automatically

5. **Switch to different project?**
   - Create new collection or reconfigure KiloCode to different collection

---

### Is indexing automatic or manual?

**Automatic!** Once configured, KiloCode handles everything:

✅ **Initial indexing** - Runs when you first open the project
✅ **File watching** - Detects changes as you code
✅ **Incremental updates** - Only re-indexes what changed
✅ **Hash-based caching** - Skips unchanged files
✅ **Git branch detection** - Detects branch switches

**You don't need to manually trigger re-indexing!**

---

### What happens when code changes or files are added?

**Smart Incremental Updates:**

- **File modified?** Only that file's embeddings are updated
- **File added?** New embeddings created and added to collection
- **File deleted?** Corresponding embeddings removed
- **Branch switch?** KiloCode detects and re-indexes if needed

**No full re-indexing required!** KiloCode uses:
- File system watching (detects changes in real-time)
- Content hashing (knows what actually changed)
- Efficient partial updates (fast, ~seconds)

**Performance:**
- Initial index: 10-20 minutes (5K-10K code blocks)
- Updates: Seconds to minutes (only changed files)

---

### What's the typical daily workflow?

**Morning (Project A):**
```
1. Open Project A in VS Code
2. KiloCode: "Using collection: project_a_codebase"
3. Work, search, code normally
4. Auto-updates happen in background
```

**Afternoon (Project B):**
```
1. Close Project A, open Project B
2. Reconfigure KiloCode collection to: project_b_codebase
3. KiloCode indexes if first time, or uses existing index
4. Work, search, code normally
```

**Key Point:** You're always working with ONE project/workspace at a time in KiloCode.

---

## Collection Configuration

### What settings do I need when creating a collection?

**Good News:** KiloCode **auto-creates collections** when you start indexing! You don't need to manually create them.

**Auto-created collection settings:**

| Setting | Value | Why |
|---------|-------|-----|
| **Collection Name** | `ws-{workspace-id}` | Auto-generated based on workspace |
| **Vector Size** | `4096` | Matches Ollama's Qwen3-Embedding-8B output |
| **Distance Metric** | `Cosine` | Optimal for semantic similarity |

**If manually creating (advanced users):**

```bash
curl -X PUT http://localhost:6333/collections/your_project_name \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 4096,
      "distance": "Cosine"
    }
  }'
```

**Optional Advanced Settings:**
- Indexing method: HNSW (default, fast search)
- Quantization: None (for maximum quality)
- On-disk storage: Enabled by default (handles large collections)

**For most users:** Just let KiloCode auto-create - it handles everything!

---

### Can I rename or delete collections?

**Yes!** Collections can be managed through:

- **Qdrant Dashboard:** http://localhost:6333/dashboard
- **API calls:** See Qdrant documentation
- **KiloCode:** Just change the collection name in settings

**Deleting a collection:**
- Permanently removes all vectors/embeddings
- Your source code is NOT affected (vectors are just indexes)
- You can always re-index by creating a new collection

**Best practice:** Use descriptive names like `my_ecommerce_app` instead of generic names like `codebase1`

---

## Troubleshooting

### My search results are from the wrong project!

**Cause:** KiloCode is pointing to the wrong collection

**Fix:**
1. Open VS Code settings
2. Navigate to KiloCode → Codebase Indexing
3. Check "Collection Name" matches your current project
4. Update if needed

---

### How do I know which collection is active?

**Check KiloCode status:**
- VS Code status bar shows indexing status
- Green = indexed and ready
- Yellow = currently indexing
- Gray = not running

**Check Qdrant Dashboard:**
- http://localhost:6333/dashboard
- View all collections and their sizes
- See which one has recent activity

---

### Do I need to backup my collections?

**Recommendation: No** (for most cases)

**Why:**
- Collections are just indexes (can be rebuilt)
- Source code is the source of truth
- Re-indexing takes 10-20 minutes (not a big deal)

**However, if you want to backup:**
- Collections are stored in Docker volume: `qdrant_storage`
- Can create snapshots via Qdrant API
- Useful for very large codebases (hours to index)

---

## Quick Reference

### Creating a Collection (Recommended: Auto)

**KiloCode auto-creates collections** - just start indexing!

1. Configure KiloCode settings
2. Click "Start Indexing"
3. Collection created automatically with correct settings

### Creating a Collection (Manual: GUI)

Only if you want custom collection names:

1. Open http://localhost:6333/dashboard
2. Click "Collections" → "New Collection"
3. Enter:
   - **Name:** `your_project_name`
   - **Vector size:** `4096`
   - **Distance:** `Cosine`
4. Click "Create"

### Creating a Collection (Manual: API)

```bash
curl -X PUT http://localhost:6333/collections/your_project_name \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 4096,
      "distance": "Cosine"
    }
  }'
```

### Configuring KiloCode

**VS Code Settings:**
1. Open Settings (Ctrl+,) or KiloCode → Codebase Indexing
2. Configure:
   - ✓ **Enable Codebase Indexing**
   - **Provider:** `Ollama`
   - **Ollama base URL:** `http://localhost:11434/`
   - **Model:** `qwen3-embedding:8b-fp16`
   - **Model dimension:** `4096`
   - **Qdrant URL:** `http://localhost:6333`
   - **Qdrant API key:** (leave empty for local)
   - **Max Results:** `50` (recommended for code, 20-25 for docs)
   - **Search score threshold:** `0.40` (default)
3. Click **Save**
4. Click **Start Indexing** (collection auto-created)

---

## Architecture Summary

```
Your Codebase (VS Code)
    ↓ (File watching)
KiloCode Extension
    ↓ (Parse with Tree-sitter)
Code Blocks (functions, classes)
    ↓ (Ollama API)
Qwen3-Embedding-8B-FP16
    ↓ (1024-dim vectors)
Qdrant Collection
    ↓ (Cosine similarity search)
Search Results (relevant code snippets)
```

**Data Flow:**
1. You write code → KiloCode detects changes
2. KiloCode sends code to Ollama → Gets embeddings
3. Embeddings stored in Qdrant collection → Ready for search
4. You search "auth logic" → Qdrant finds similar vectors
5. Results displayed in KiloCode → Jump to code

---

## Next Steps

1. ✅ Qdrant running (check dashboard)
2. ⏭️ Create your first collection (`kilocode_codebase` for testing)
3. ⏭️ Configure KiloCode to use that collection
4. ⏭️ Let it index your codebase
5. ⏭️ Try semantic searches!

---

**Need more help?** Check the detailed guides:
- [qdrant-setup-guide.md](qdrant-setup-guide.md) - Step-by-step Qdrant deployment
- [kilocode-codebase-indexing-docs.md](kilocode-codebase-indexing-docs.md) - KiloCode feature documentation
- [qwen3-embedding-kilocode-setup-report.md](qwen3-embedding-kilocode-setup-report.md) - Why Qwen3 and technical details
