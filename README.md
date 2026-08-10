# Local RAG — Personal Knowledge Base for AI Agents

A self-hosted Chroma-based knowledge base skill for AI agents. Store documents, notes, and files locally, then retrieve them via semantic search.

## What This Does

- **Ingest** any file format into a Chroma vector database (20+ formats)
- **Query** your personal knowledge base with natural language
- **Manage** collections: create, list, rename, purge, delete
- **Embed** with [embeddinggemma-300m](https://www.modelscope.cn/google/embeddinggemma-300m) (768-dim, runs locally, no API key needed)

Supported formats: `.md` `.txt` `.pdf` `.docx` `.pptx` `.xlsx` `.csv` `.html` `.json` `.yaml` `.epub` `.rtf` `.msg` `.ipynb` `.odt` plus images (OCR) and audio (Whisper transcription).

---

## Quick Start

### Step 1: Deploy Chroma

```bash
# Create a directory for Chroma
mkdir -p ~/chroma && cd ~/chroma

# Write docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  chroma:
    image: chromadb/chroma:latest
    container_name: chroma
    restart: unless-stopped
    ports:
      - "127.0.0.1:8100:8000"
    volumes:
      - ./chroma-data:/chroma/.chroma
    environment:
      - ANONYMIZED_TELEMETRY=False
      - ALLOW_RESET=True
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/api/v2/heartbeat"]
      interval: 10s
      timeout: 3s
      retries: 5
EOF

# Start Chroma
docker compose up -d

# Verify it's running
curl -fsS http://127.0.0.1:8100/api/v2/heartbeat
```

> **Port conflict?** If port 8100 is taken, change both the host port and `CHROMA_PORT` env var below. Check with: `ss -ltnp | grep ':8100'`

### Step 2: Install Python Dependencies

```bash
# Core (required)
pip install chromadb

# File format support (install as needed)
pip install pypdf            # PDF reading
pip install python-docx      # Word documents
pip install python-pptx      # PowerPoint
pip install openpyxl         # Excel
pip install PyYAML           # YAML
pip install ebooklib         # EPUB
pip install odfpy            # OpenDocument
pip install striprtf         # RTF
pip install extract-msg      # Outlook .msg

# Media support (install as needed)
pip install openai-whisper   # Audio transcription
pip install pytesseract Pillow  # Image OCR
# Also requires: apt install tesseract-ocr tesseract-ocr-chi-sim

# PDF OCR fallback (for image-based PDFs)
pip install pymupdf
```

### Step 3: Install the Skill

Clone this repo and copy to your agent's skill directory:

```bash
# Clone the repo
git clone https://github.com/haoyiyin/local-rag.git /tmp/local-rag

# Install to the correct path based on your agent:

# Hermes Agent
cp -r /tmp/local-rag ~/.hermes/skills/mlops/local-rag

# OpenHands / Agents
# Copy to your agent's skills/ directory

# Or just place SKILL.md + scripts/ wherever your agent loads skills from
```

### Step 4: Download the Embedding Model

The first time you run a command, the embedding model (~80 MB) will be downloaded automatically. You can also pre-download:

```bash
# The model is cached at:
# ~/.cache/modelscope/models/google--embeddinggemma-300m/snapshots/master
# It downloads on first use — no manual step needed.
```

### Step 5: Create Your First Knowledge Base

```bash
# List existing collections
python3 scripts/rag.py list

# Create a new collection (you'll be prompted for a name)
python3 scripts/rag.py create my-notes

# Or create with metadata
python3 scripts/rag.py create work-docs --meta description="Work documents" category="professional"
```

### Step 6: Start Using It

```bash
# Ingest a file
python3 scripts/rag.py ingest /path/to/document.pdf --tag notes

# Ingest text from stdin
echo "Important note: remember to..." | python3 scripts/rag.py ingest - --tag quick-note

# Query your knowledge base
python3 scripts/rag.py ask "What did I note about quarterly targets?"

# Check collection details
python3 scripts/rag.py info my-notes
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CHROMA_HOST` | `127.0.0.1` | Chroma server host |
| `CHROMA_PORT` | `8100` | Chroma server port |
| `KB_COLLECTION` | `default` | Default collection name |
| `KB_EMBED_MODEL` | `~/.cache/modelscope/.../embeddinggemma-300m` | Embedding model path |

---

## CLI Commands Reference

### Collection Management

| Command | Description |
|---------|-------------|
| `list` | List all collections with record counts |
| `create <name>` | Create a new collection (uses embeddinggemma-300m) |
| `delete <name>` | Delete a collection and all its data |
| `info <name>` | Show collection details and sample records |
| `rename <old> <new>` | Rename a collection |
| `purge <name>` | Delete all records in a collection |

### Data Operations

| Command | Description |
|---------|-------------|
| `ingest <path>` | Store a file (supports 20+ formats) |
| `ingest-text "text"` | Store a text string directly |
| `ask "query"` | Semantic search with natural language |
| `smoke` | Test connectivity to Chroma |

All data commands accept `-c <collection>` to target a specific collection.

---

## Agent Installation Guide

This skill is designed to be installed by AI agents. **Check each dependency before installing — skip if already present.**

### Pre-flight Checks

```bash
# 1. Chroma running?
if curl -fsS http://127.0.0.1:8100/api/v2/heartbeat >/dev/null 2>&1; then
  echo "Chroma OK — skip deploy"
else
  echo "Chroma not running — need docker compose deploy"
fi

# 2. Docker available? (only needed if deploying Chroma)
docker --version 2>/dev/null || echo "Docker missing — ask user to install"

# 3. Embedding model cached?
if [ -d "$HOME/.cache/modelscope/models/google--embeddinggemma-300m" ]; then
  echo "Model OK — skip download"
else
  echo "Model not cached — will auto-download on first use (~80MB)"
fi

# 4. Python deps (import-check each, install only missing)
python3 -c "import chromadb" 2>/dev/null && echo "chromadb OK" || echo "pip install chromadb"
python3 -c "import pypdf" 2>/dev/null && echo "pypdf OK" || echo "pip install pypdf"
python3 -c "import docx" 2>/dev/null && echo "python-docx OK" || echo "pip install python-docx"
python3 -c "import pptx" 2>/dev/null && echo "python-pptx OK" || echo "pip install python-pptx"
python3 -c "import openpyxl" 2>/dev/null && echo "openpyxl OK" || echo "pip install openpyxl"
python3 -c "import yaml" 2>/dev/null && echo "PyYAML OK" || echo "pip install pyyaml"
```

### Install Steps (skip what's already there)

1. **Chroma** — `curl -fsS http://127.0.0.1:8100/api/v2/heartbeat` responds → skip. Otherwise deploy (see Step 1).
2. **Embedding model** — `ls ~/.cache/modelscope/models/google--embeddinggemma-300m` exists → skip. Otherwise auto-downloads on first use.
3. **Python packages** — import-check each. Only `pip install` missing ones.
4. **Copy skill** to `~/.hermes/skills/mlops/local-rag/` (Hermes) or your agent's skill dir.
5. **Ask user** for their first collection name.
6. **Create collection** and run `smoke` to verify.

---

## License

MIT
