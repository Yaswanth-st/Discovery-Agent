# Discovery Agent

> Autonomous AI agent that reads mixed-format company documents, extracts a confidence-scored system inventory, maps integration gaps, and generates production-ready connector code — powered by Groq (free API).

---

## What It Does

Most enterprises have no accurate map of the software systems they actually use. Information is scattered across PDFs, wikis, spreadsheets, and tribal knowledge. This agent reads through those documents and builds that map automatically — then tells you what's missing and generates the code to fix it.

**Three progressive levels:**

| Level | What it does |
|-------|-------------|
| **L1 — Discovery** | Reads documents, extracts every system with confidence scores and evidence chains |
| **L2 — Gap Analysis** | Maps automation goals to systems, identifies missing integrations, ranks by business impact |
| **L3 — Code Generation** | Generates runnable Python connector modules, YAML agent definitions, unit tests, and READMEs |

---

## Architecture

```
Documents (PDF, PNG/JPG, TXT, MD, HTML, XLSX, CSV)
    ↓
Format Router — specialist parser per file type
    ↓
Multi-Pass Extractor (3 focused LLM calls per document)
    Pass 1: Explicitly named systems
    Pass 2: Implied / inferred systems
    Pass 3: Evidence verification — hallucination guard
    ↓
Rules-Based Confidence Scorer (not LLM-assigned)
    ↓
Deduplication + Alias Resolution
    ↓
Level 2: Use Case Mapper + Integration Gap Ranker
    ↓
Level 3: Connector Code Generator (Python + YAML + Tests)
    ↓
Outputs: JSON · Markdown · SQLite history · Audit log
```

**Key design decision:** Confidence scores are computed from evidence rules, not from what the LLM claims. The LLM extracts and cites; the rules engine decides how much to trust each finding. This eliminates the most common failure mode in LLM-based extraction — fabricated confidence.

---

## Project Structure

```
discovery_agent/
├── app.py                    # Streamlit UI — all 3 levels in one interface
├── main.py                   # CLI entry point
├── models.py                 # Pydantic data models (typed, validated)
├── audit.py                  # Structured event logging
├── storage.py                # SQLite persistence for run history
├── level1_service.py         # Level 1 pipeline orchestration
├── gap_analysis.py           # Level 2 use case mapping and gap detection
├── connector_generator.py    # Level 3 code generation
├── diagnose.py               # API key / environment diagnostics
│
├── extractor/
│   └── extractor.py          # Multi-pass LLM extractor with hallucination guard
│
├── loaders/
│   └── loader.py             # Format-specific document loaders
│
├── validator/
│   └── validator.py          # Rules-based confidence scoring
│
├── output/
│   └── writer.py             # JSON and Markdown report writer
│
├── sample_docs/              # Sample documents to demo the agent
│   ├── onboarding_wiki.md
│   ├── it_system_inventory.csv
│   ├── integration_audit.txt
│   └── automation_goals.md
│
├── tests/
│   ├── test_validator.py
│   ├── test_loader.py
│   ├── test_gap_analysis.py
│   └── test_connector_generator.py
│
├── requirements.txt
├── .env.example
└── README.md
```

---

## Setup

### 1. Clone and install

```bash
git clone https://github.com/Yaswanth-st/Discovery-Agent.git
cd Discovery-Agent/discovery_agent/discovery_agent
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
```

Open `.env` and add your Groq API key:

```
GROQ_API_KEY=your_groq_api_key_here
```

Get a free key at [console.groq.com](https://console.groq.com) — no credit card required.

### 3. (Optional) Install Tesseract for image/scanned PDF OCR

```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr

# macOS
brew install tesseract

# Windows — download installer from:
# https://github.com/UB-Mannheim/tesseract/wiki
```

---

## Running the Agent

### Option A — Streamlit UI (recommended)

```bash
streamlit run app.py
```

Opens at `http://localhost:8501`. Provides the full Level 1 → 2 → 3 workflow with document upload, live results, persistent history, and artifact downloads.

### Option B — CLI

```bash
# Basic run
python main.py --docs ./sample_docs --output ./output

# Interactive terminal UI (step-by-step demo mode)
python main.py --interactive --docs ./sample_docs --output ./output
```

### Option C — Diagnose environment

```bash
python diagnose.py
```

Checks API key, model availability, and library versions before you run.

---

## Supported Document Formats

| Format | How it's loaded |
|--------|----------------|
| `.pdf` | Text layer extraction via PyMuPDF; OCR fallback via Tesseract |
| `.png`, `.jpg`, `.jpeg`, `.webp` | OCR via Tesseract + Pillow |
| `.txt`, `.md` | Direct text read |
| `.html`, `.htm` | Tag-stripped via BeautifulSoup |
| `.xlsx`, `.xls` | Tabular extraction via openpyxl / pandas |
| `.csv` | Row-by-row via pandas |

---

## How the Extractor Works

Each document goes through **three focused LLM passes** using `llama-3.3-70b-versatile` on Groq:

**Pass 1 — Explicit extraction**
Finds systems named directly: "We use Salesforce for CRM" → `Salesforce`.

**Pass 2 — Inferred extraction**
Finds implied systems: "Our deals pipeline" near "SFDC" → `Salesforce` (alias resolved).

**Pass 3 — Verification (hallucination guard)**
Re-reads the raw document text and confirms each candidate from passes 1 and 2 is genuinely supported by a direct quote. Unverified candidates are penalised in confidence scoring, not silently accepted.

---

## Confidence Scoring

Scores are computed deterministically from evidence signals — not from the LLM's self-reported confidence:

| Signal | Points |
|--------|--------|
| Verified quote found in source | +40 |
| Explicit naming (not inferred) | +20 |
| Found in multiple documents | +15 |
| Key entities identified | +10 |
| Business processes identified | +10 |
| Criticality stated | +5 |
| Inferred with no verification | −20 |
| Zero verified mentions | −10 |

| Score | Tier | Meaning |
|-------|------|---------|
| 95–100% | High | Explicitly named with full context |
| 70–94% | Medium | Named but limited context |
| < 70% | Low | Flagged for human review |

Systems below 70% are flagged with `human_review_required: true` and include a `review_reason` explaining exactly what was inferred and what evidence was missing.

---

## Output Files

| File | Description |
|------|-------------|
| `output/discovery_result.json` | Structured JSON — one entry per system, full evidence chains |
| `output/discovery_report.md` | Human-readable Markdown report with evidence citations |
| `output/audit_log.jsonl` | Structured event trail — every load, extraction, verification, and scoring decision |
| `output/ui/discovery_agent.sqlite3` | Persistent run history across sessions |
| `output/ui/generated_connectors/<name>/` | Generated connector package per integration gap |

---

## Level 2 — Gap Analysis

Paste or upload a list of automation goals in plain English. The agent:

1. Maps each goal to the systems it requires
2. Traces source → destination data flows with trigger events
3. Checks which system-to-system integrations exist vs. are missing
4. Ranks gaps by priority score (frequency × affected use cases × criticality)
5. Estimates implementation effort for each gap

**Example input:**
```
- Finance wants Expensify expenses to sync into NetSuite every night.
- Engineering wants GitHub pull requests linked to Jira tickets with Slack updates.
- Marketing leads from HubSpot should flow into Salesforce with deduplication.
```

---

## Level 3 — Connector Code Generation

For each prioritised gap, the agent generates a complete connector package:

```
generated_connectors/
└── stripe-netsuite/
    ├── connector.py          # Auth, CRUD, pagination, retries, error handling
    ├── agent.yaml            # Agent definition — system prompt, tools, workflow steps
    ├── tests/
    │   └── test_connector.py # Unit tests — auth, CRUD, error conditions, edge cases
    ├── requirements.txt
    └── README.md             # Setup instructions and env var config
```

Generated code targets **70–80% production readiness** — an engineer reviewing it should need to change fewer than 10% of lines before deploying.

---

## Tests

```bash
python -m pytest tests/ -v
```

Tests cover confidence scoring, alias resolution, hallucination detection, document loading, gap analysis logic, and connector code generation. **No API key required.**

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| LLM | Groq API — `llama-3.3-70b-versatile` (free tier) |
| UI | Streamlit |
| Data models | Pydantic v2 |
| PDF extraction | PyMuPDF |
| OCR | Tesseract + Pillow |
| Spreadsheets | openpyxl + pandas |
| HTML parsing | BeautifulSoup4 |
| Persistence | SQLite |
| CLI formatting | Rich |
| Observability | JSONL audit log |

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GROQ_API_KEY` | Yes | Free API key from console.groq.com |

---

## Submission — Aivar Innovations Hiring Challenge

Agent 3: Discovery Agent — Levels 1, 2, and 3.
**Candidate:** Yaswanth S T · 23BCS191

| Deliverable | Link |
|-------------|------|
| Code | [github.com/Yaswanth-st/Discovery-Agent](https://github.com/Yaswanth-st/Discovery-Agent) |
| Video | https://drive.google.com/file/d/1aGNw9LjUe4FT-R7VoVNlbAjPT4dapPtb/view?usp=sharing |
| Write-up | https://drive.google.com/file/d/1ep_nTuEQnvDYJ1JLT5a-Y4zwBD7G5ORD/view?usp=sharing |
