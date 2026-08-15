# Local RAG — Personal Knowledge Base for AI Agents

A local Chroma-based knowledge base skill for AI agents. Store documents, notes, and files, then retrieve them via semantic search. No Docker, no server — runs directly on your machine.

## What This Does

- **Ingest** any file format into a local Chroma database (20+ formats)
- **Query** your personal knowledge base with natural language
- **Manage** collections: create, list, rename, purge, delete
- **Embed** with [embeddinggemma-300m](https://www.modelscope.cn/google/embeddinggemma-300m) — default loads the **Q8_0 GGUF (~334MB, 真量化)** via llama.cpp; 768-dim, runs locally, no API key needed

Supported formats: `.md` `.txt` `.pdf` `.docx` `.pptx` `.xlsx` `.csv` `.html` `.json` `.yaml` `.epub` `.rtf` `.msg` `.ipynb` `.odt` plus images (OCR) and audio (Whisper transcription).

---

## Quick Start

### Step 1: Install Dependencies

```bash
# Required
pip install chromadb
pip install llama-cpp-python

# Optional — install based on what formats you need
pip install pypdf            # PDF
pip install python-docx      # Word
pip install python-pptx      # PowerPoint
pip install openpyxl         # Excel
pip install PyYAML           # YAML
pip install ebooklib         # EPUB
pip install openai-whisper   # Audio
pip install pytesseract Pillow  # Image OCR (+ apt install tesseract-ocr)
pip install pymupdf          # PDF OCR fallback
```

### Step 2: Install the Skill

```bash
git clone https://github.com/haoyiyin/local-rag.git /tmp/local-rag

# Hermes Agent
cp -r /tmp/local-rag ~/.hermes/skills/mlops/local-rag

# Other agents: copy to your skills directory
```

### Step 3: Create Your First Knowledge Base

```bash
# List existing collections
python3 scripts/rag.py list

# Create a new collection
python3 scripts/rag.py create my-notes
```

### Step 4: Use It

```bash
# Ingest a file
python3 scripts/rag.py ingest /path/to/document.pdf --tag notes

# Ingest text
echo "Important note: ..." | python3 scripts/rag.py ingest - --tag quick-note

# Query
python3 scripts/rag.py ask "What did I note about quarterly targets?"

# Collection details
python3 scripts/rag.py info my-notes
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CHROMA_PATH` | `~/.chroma/local-rag` | Local data storage path |
| `KB_COLLECTION` | `default` | Default collection name |
| `KB_EMBED_MODEL` | *(auto)* Q8 GGUF cache path | Embedding model. Set to a sentence-transformers model/dir to override |

---

## CLI Commands Reference

### Collection Management

| Command | Description |
|---------|-------------|
| `list` | List all collections with record counts |
| `create <name>` | Create a new collection (uses embeddinggemma-300m Q8_0 GGUF) |
| `delete <name>` | Delete a collection and all its data |
| `info <name>` | Show collection details and sample records |
| `rename <old> <new>` | Rename a collection |
| `purge <name>` | Delete all records in a collection |

### Data Operations

| Command | Description |
|---------|-------------|
| `ingest <path>` | Store a file or archive (see supported formats below) |
| `ingest-text "text"` | Store a text string directly |
| `ask "query"` | Semantic search with natural language |
| `smoke` | Test connectivity |

**Supported file formats:**
`.md` `.txt` `.pdf` `.docx` `.pptx` `.xlsx` `.csv` `.html` `.json` `.yaml` `.epub` `.rtf` `.msg` `.ipynb` `.odt` plus images (OCR) and audio (Whisper).

**Supported archives** (auto-detected, extracts all files inside):
`.zip` `.tar` `.tar.gz` `.tgz` `.tar.bz2` `.tbz2` `.tar.xz` `.txz` `.7z` `.rar`

When an archive is uploaded, it automatically extracts all text files inside (skipping binary files like images/audio/video) and ingests them individually. Each file's source is recorded as `archive_name/filename`.

All data commands accept `-c <collection>` to target a specific collection.

---

## Agent Installation Guide

This section is for AI agents installing this skill. **Do not install anything that already exists.**

### Phase 1: Check Everything First

Run ALL of these checks before installing anything. Collect the results.

```bash
# Python + chromadb (required)
python3 -c "import chromadb" 2>/dev/null && echo "chromadb=ok" || echo "chromadb=missing"

# Embedding model — Q8 GGUF, auto-downloaded on first use
python3 -c "import llama_cpp" 2>/dev/null && echo "llama-cpp=ok" || echo "llama-cpp=missing"
[ -f "$HOME/.cache/modelscope/models/ggml-org--embeddinggemma-300M-GGUF/snapshots/master/embeddinggemma-300M-Q8_0.gguf" ] && echo "MODEL=ok" || echo "MODEL=missing"

# Optional sentence-transformers backend (only when KB_EMBED_MODEL is set)
python3 -c "import sentence_transformers" 2>/dev/null && echo "sentence-transformers=ok" || echo "sentence-transformers=missing"

# Python packages (optional, install based on user's format needs)
python3 -c "import pypdf" 2>/dev/null && echo "pypdf=ok" || echo "pypdf=missing"
python3 -c "import docx" 2>/dev/null && echo "python-docx=ok" || echo "python-docx=missing"
python3 -c "import pptx" 2>/dev/null && echo "python-pptx=ok" || echo "python-pptx=missing"
python3 -c "import openpyxl" 2>/dev/null && echo "openpyxl=ok" || echo "openpyxl=missing"
python3 -c "import yaml" 2>/dev/null && echo "pyyaml=ok" || echo "pyyaml=missing"
python3 -c "import ebooklib" 2>/dev/null && echo "ebooklib=ok" || echo "ebooklib=missing"
python3 -c "import whisper" 2>/dev/null && echo "openai-whisper=ok" || echo "openai-whisper=missing"
python3 -c "import pytesseract" 2>/dev/null && echo "pytesseract=ok" || echo "pytesseract=missing"
python3 -c "import fitz" 2>/dev/null && echo "pymupdf=ok" || echo "pymupdf=missing"
```

### Phase 2: Install Only What's Missing

Based on Phase 1 results, install only items marked `missing`. **Skip everything marked `ok`.**

**If chromadb=missing:**
- `pip install chromadb`

**If llama-cpp=missing:**
- `pip install llama-cpp-python` (required — default Q8 GGUF backend)

**If MODEL=missing:**
- No action needed. `rag.py` auto-downloads `embeddinggemma-300M-Q8_0.gguf` from ModelScope on first use (~334MB, 1-3 min). Warn the user.

**If sentence-transformers=missing:**
- Only needed if user sets `KB_EMBED_MODEL` to a PyTorch sentence-transformers model. Skip otherwise.

**If any other Python package=missing:**
- `pip install <package-name>` for each one

### Phase 3: Install the Skill

Copy to the correct path based on your agent:

| Agent | Path |
|---|---|
| Hermes Agent | `~/.hermes/skills/mlops/local-rag/` |
| Other | Your agent's skills directory |

### Phase 4: Create First Knowledge Base

**Ask the user:** "What would you like to name your knowledge base?"

Then run: `python3 scripts/rag.py create <user-provided-name>`

After creating, tell the user what they can upload:
- **Files:** `.md` `.txt` `.pdf` `.docx` `.pptx` `.xlsx` `.csv` `.html` `.json` `.yaml` `.epub` `.rtf` `.msg` `.ipynb` `.odt` plus images and audio
- **Archives:** `.zip` `.tar.gz` `.7z` `.rar` and other tar variants — auto-extracts all files inside

Confirm: `python3 scripts/rag.py smoke`

---

## License

MIT
