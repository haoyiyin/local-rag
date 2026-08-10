---
name: local-rag
description: "Local Chroma: full API reference plus ingest/ask CLI."
version: 2.0.0
author: Hermes Agent session
license: MIT
metadata:
  hermes:
    tags: [rag, chroma, knowledge-base, retrieval, vector-db, local]
    related_skills: [local-vector-db-deploy]
---

# Local RAG — Chroma Full API Reference + Personal Knowledge Base

Two layers: (1) `scripts/rag.py` CLI for quick ingest/ask, (2) full Chroma Python API reference below.

## When to load

- User wants to store/retrieve from personal knowledge base ("记住", "存到知识库", "search my notes")
- User needs to work with Chroma API directly (collections, queries, filters, CRUD)
- User is debugging or configuring local Chroma
- Skip: general web search, debugging Chroma deployment → `local-vector-db-deploy`, reading a file → file tools

## Connection

Data stored locally at `~/.chroma/local-rag` (PersistentClient, no server needed).

```
CHROMA_PATH=~/.chroma/local-rag  KB_COLLECTION=default
```

`scripts/rag.py` reads these env vars. Embedding model: `embeddinggemma-300m` (768-dim, modelscope path).

---

## Part 1: CLI (scripts/rag.py)

### Collection management

```bash
# List all collections (with counts)
python3 scripts/rag.py list [--limit N] [--offset N]

# Create collection (auto-selects embeddinggemma-300m)
python3 scripts/rag.py create <name> [--meta key1=val1 key2=val2]

# Delete collection (⚠️ all data lost)
python3 scripts/rag.py delete <name>

# Collection details (count + metadata + sample records)
python3 scripts/rag.py info <name> [--sample 3]

# Rename collection
python3 scripts/rag.py rename <old_name> <new_name>

# Purge all records (keeps collection empty)
python3 scripts/rag.py purge <name>
```

All output is JSON. `list` returns `[{name, id, count, metadata}]`.

**Creating a new knowledge base:** When user asks to create a new KB, ask for the collection name, then run `create`. Default embedding model is always `embeddinggemma-300m` (768-dim, via modelscope). Do not ask about embedding model unless user explicitly wants a different one.

### Data commands

All data commands support `-c <collection>` to target a specific collection (default: `$KB_COLLECTION` or `default`).

#### ingest — store files

```bash
python3 scripts/rag.py ingest <path> [--tag TAG] [--source LABEL] [-c COLLECTION]
echo "text" | python3 scripts/rag.py ingest - [--tag TAG] [-c COLLECTION]
python3 scripts/rag.py ingest-text "text" [--source LABEL] [--tag TAG] [-c COLLECTION]
```

Supported formats: `.md .txt .pdf .mp3/.wav/.ogg/.flac/.m4a/.aac/.wma/.opus` (whisper) `.png/.jpg/.jpeg/.gif/.bmp/.tiff/.tif/.webp` (OCR) `.docx .pptx .xlsx/.xls .csv .html/.htm .json/.jsonl .yaml/.yml .odt .epub .rtf .msg .ipynb`. Others: plain text.

Behavior: splits on blank lines, cap ~1200 chars/chunk. ID = `sha1(source+index)[:16]` → idempotent. PDF: pypdf first, OCR fallback for image PDFs. Audio: whisper base model CPU. Images: pytesseract chi_sim+eng.

#### ask — retrieve

```bash
python3 scripts/rag.py ask "<question>" [--k 5] [--tag TAG] [-c COLLECTION]
```

Returns JSON: `[{source, tag, chunk, score}]`. Score = distance (lower = more similar). Threshold `< 1.2` ≈ citable.

#### smoke — connectivity test

```bash
python3 scripts/rag.py smoke
```

---

## Part 2: Chroma Python API Reference

### Client Setup

```python
import chromadb

# Local persistent client (default — no server needed)
client = chromadb.PersistentClient(path="~/.chroma/local-rag")

# Async not available for PersistentClient; use sync API
```

### Client Methods

#### Collection CRUD

```python
# Create collection (fails if exists)
col = client.create_collection("my_col", metadata={"description": "test"})

# Get existing collection (fails if not exists)
col = client.get_collection("my_col")

# Create or get (idempotent)
col = client.get_or_create_collection("my_col")

# Get collection by UUID
col = client.get_collection_by_id(uuid.UUID("..."))

# List all collections (paginated)
cols = client.list_collections(limit=10, offset=0)

# Count collections
n = client.count_collections()

# Delete collection
client.delete_collection("my_col")
```

#### Collection Configuration (create)

```python
col = client.create_collection(
    "my_col",
    metadata={"description": "notes"},
    configuration={
        "hnsw:space": "cosine",           # l2 | cosine | ip
        "hnsw:ef_construction": 100,       # index build quality
        "hnsw:ef_search": 100,             # query quality (modifiable)
        "hnsw:max_neighbors": 16,          # graph density
        "hnsw:num_threads": 4,             # parallel threads
        "hnsw:batch_size": 100,            # indexing batch size
        "hnsw:sync_threshold": 1000,       # WAL sync threshold
        "hnsw:resize_factor": 1.2,         # growth factor
    }
)
```

#### Other Client Methods

```python
client.heartbeat()             # → int (ns timestamp)
client.get_version()           # → str ("0.6.x")
client.get_settings()          # → Settings object
client.get_user_identity()     # → UserIdentity
client.get_max_batch_size()    # → int
client.clear_system_cache()    # clear system cache
client.reset()                 # → bool (⚠️ DELETES ALL DATA)
client.close()                 # close connection
```

#### Multi-tenant

```python
client.set_tenant("tenant_a")
client.set_tenant("tenant_b", database="db2")
```

---

### Collection Methods

#### add — insert records

```python
col.add(
    ids=["id1", "id2"],
    documents=["hello world", "foo bar"],      # optional (Chroma embeds)
    embeddings=[[0.1, 0.2, ...], [0.3, 0.4]],  # optional (pre-computed)
    metadatas=[{"source": "file.txt", "tag": "note"}, {"source": "web", "tag": "clip"}],
    images=[numpy_array, ...],                  # optional (multi-modal)
    uris=["/path/to/image.png", ...],           # optional (lazy load)
)
```

- IDs must be unique; duplicate IDs → ignored (no error)
- `documents` OR `embeddings` required; both → error
- `metadatas`: values = str|int|float|bool, or arrays of same type

#### upsert — insert or overwrite

```python
col.upsert(
    ids=["id1"],
    documents=["updated text"],
    metadatas=[{"version": 2}],
)
```

Same as `add` but overwrites if ID exists.

#### update — modify existing records

```python
col.update(
    ids=["id1"],
    documents=["new text"],
    metadatas=[{"updated": True}],
    embeddings=[[0.5, 0.6, ...]],  # optional
)
```

Only updates provided fields. ID must exist.

#### get — retrieve records (no vector search)

```python
result = col.get(
    ids=["id1", "id2"],                    # optional: specific IDs
    where={"tag": "note"},                  # optional: metadata filter
    where_document={"$contains": "hello"},  # optional: text filter
    limit=10,                               # optional: max results
    offset=0,                               # optional: pagination
    include=["documents", "metadatas", "embeddings"],  # optional
)
# result.ids, result.documents, result.metadatas, result.embeddings
```

#### peek — first N records

```python
result = col.peek(limit=10)  # convenience get with default offset=0
```

#### query — vector similarity search

```python
result = col.query(
    query_texts=["search query"],             # text (auto-embedded)
    query_embeddings=[[0.1, 0.2, ...]],       # or pre-computed vectors
    query_images=[numpy_array],               # or image (multi-modal)
    query_uris=["/path/to/image.png"],        # or URI
    n_results=10,                              # top-k
    ids=["id1", "id2"],                        # optional: restrict to IDs
    where={"tag": "note"},                     # optional: metadata filter
    where_document={"$contains": "hello"},     # optional: text filter
    include=["documents", "metadatas", "distances", "embeddings"],  # optional
)
# result.ids[0], result.documents[0], result.distances[0], result.metadatas[0]
```

- Batch: multiple queries in one call → `query_texts=["q1", "q2"]`
- `where`, `where_document`, `ids` applied to all queries
- `distances`: lower = more similar

#### delete — remove records

```python
# By IDs
col.delete(ids=["id1", "id2"])

# By metadata filter
col.delete(where={"tag": "old"})

# By document content filter
col.delete(where_document={"$contains": "deprecated"})

# By both + limit
col.delete(where={"tag": "old"}, limit=50)
```

#### count — record count

```python
n = col.count()
# read_level options: "index_and_wal" (default), "index_only", "index_and_bounded_wal"
```

#### modify — rename / reconfigure

```python
col.modify(
    name="new_name",
    metadata={"description": "updated"},
    configuration={"hnsw:ef_search": 200},  # update HNSW params
)
```

#### fork — clone collection (Chroma Cloud only)

```python
new_col = col.fork("my_col_copy")
n = col.fork_count()
```

#### Attached Functions (Chroma Cloud)

```python
from chromadb.api.functions import STATISTICS_FUNCTION
fn, created = col.attach_function(
    function=STATISTICS_FUNCTION,
    name="my_stats",
    output_collection="my_stats_output",
)
col.detach_function("my_stats", delete_output_collection=True)
fn = col.get_attached_function("my_stats")
```

#### Indexing Status

```python
status = col.get_indexing_status()
# status.num_indexed_ops, status.num_unindexed_ops, status.total_ops, status.op_indexing_progress
```

---

### Metadata Filtering

```python
# Equality
where = {"tag": "note"}

# Comparison operators
where = {"page": {"$gt": 10}}      # greater than
where = {"page": {"$gte": 10}}     # greater or equal
where = {"page": {"$lt": 5}}       # less than
where = {"page": {"$lte": 5}}      # less or equal
where = {"page": {"$ne": 3}}       # not equal
where = {"page": {"$eq": 3}}       # explicit equal

# Inclusion
where = {"author": {"$in": ["alice", "bob"]}}
where = {"author": {"$nin": ["charlie"]}}

# Array metadata
where = {"tags": {"$contains": "important"}}
where = {"tags": {"$not_contains": "archived"}}

# Logical: $and / $or
where = {"$and": [{"tag": "note"}, {"page": {"$gt": 5}}]}
where = {"$or": [{"color": "red"}, {"color": "blue"}]}

# Nested logical
where = {"$and": [
    {"$or": [{"tag": "a"}, {"tag": "b"}]},
    {"page": {"$gte": 10}},
]}
```

### Full Text Search (where_document)

```python
where_doc = {"$contains": "hello"}         # case-sensitive substring
where_doc = {"$not_contains": "test"}
where_doc = {"$regex": r"\b\w+@\w+\.\w+\b"}  # regex
where_doc = {"$not_regex": r"spam"}

# Logical
where_doc = {"$and": [{"$contains": "hello"}, {"$not_contains": "world"}]}
where_doc = {"$or": [{"$contains": "foo"}, {"$contains": "bar"}]}
```

Combine `where` + `where_document` in `get()` and `query()`.

---

### HNSW Distance Metrics

| Metric | param | Equation | Use case |
|--------|-------|----------|----------|
| Squared L2 | `l2` | `Σ(Ai-Bi)²` | Spatial proximity |
| Inner Product | `ip` | `1.0 - Σ(Ai×Bi)` | Recommendation systems |
| Cosine | `cosine` | `1.0 - Σ(Ai×Bi) / √(ΣAi²)·√(ΣBi²)` | Text embeddings (default for most models) |

---

### Conditional Transactions

```python
# Manual transaction
txn = col.transaction()
records = txn.get(where={"tag": "draft"})
for r in records["ids"]:
    txn.update(ids=[r], metadatas=[{"status": "published"}])
txn.commit()

# Auto-retry transaction
def publish(txn):
    records = txn.get(where={"tag": "draft"})
    for r in records["ids"]:
        txn.update(ids=[r], metadatas=[{"status": "published"}])
col.transaction().run(publish)
```

Limitations: collection-scoped, no nested, no `query()` in txn, explicit IDs for delete, max 1 write per ID per txn.

---

## How Hermes Uses This

### ask → synthesize

1. Run `scripts/rag.py ask "<query>" --k 5`
2. If top chunk `score < 1.2`: fold top 3 into context as `[From my notes — <source>]\n<chunk>`
3. Cite: `(your notes: <source>)`
4. If no match: answer normally, say "no relevant notes found"

### ingest → store

1. File: `scripts/rag.py ingest <path>`
2. Text: `echo "..." | scripts/rag.py ingest -`
3. Confirm: `saved as <source>`

### Direct API usage

When user asks for complex queries (filters, pagination, batch ops, metadata management), use Python directly:

```python
import chromadb
c = chromadb.PersistentClient(path=os.path.expanduser("~/.chroma/local-rag"))
col = c.get_or_create_collection("default")
# ... direct API calls
```

---

## Hard Rules

- **No LLM in scripts.** ingest = embed+store. ask = retrieve. Synthesis = Hermes.
- **ask is read-only.** Never writes.
- **Surface failures.** Exit non-zero with error message.
- **Deterministic IDs.** sha1-based, idempotent re-ingest overwrites.
- **v2 API only.** No `Settings(anonymized_telemetry=...)` — that's v1.

## Pitfalls

- **First call slow (~20s).** Embedding model download. Not a hang.
- **Score = distance, not similarity.** Lower = better. Threshold 1.2 tuned for embeddinggemma-300m.
- **Long single-line paste.** `ingest -` on 50k chars = one giant chunk. Bad for transcripts.
- **`add()` ignores duplicate IDs silently.** Use `upsert()` to overwrite.
- **Metadata values:** only str|int|float|bool and arrays. No nested dicts.
- **where_document is case-sensitive.**
- **`reset()` deletes everything.** Never call without explicit user intent.
