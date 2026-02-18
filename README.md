# 🤖 AI-Driven SQL Ingestion Pipeline

> An intelligent, end-to-end data ingestion pipeline that uses **OpenAI GPT-4o** and **Milvus vector search** to automatically parse Excel/CSV files, infer schema, detect duplicates, and load structured data into **PostgreSQL** — with a human-in-the-loop approval workflow.

![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-green?logo=fastapi)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-14+-blue?logo=postgresql)
![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o-orange?logo=openai)
![Milvus](https://img.shields.io/badge/Milvus-Vector_DB-purple)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## ✨ Features

| Feature | Description |
|---|---|
| 🤖 **LLM-Powered Schema Inference** | GPT-4o analyzes file structure, detects multi-row headers, infers column types, and generates standardized table names |
| 🔄 **Incremental Load (IL)** | Milvus vector similarity search matches new files to existing tables for automatic data appending |
| 🔍 **Duplicate Detection** | Hybrid date parsing (`dateutil`) compares period values to prevent re-uploading the same data |
| 🧩 **Schema Validation** | Automatically maps column name mismatches (e.g., `baseyear` → `base_year`) before insertion |
| ✅ **Human-in-the-Loop** | Web UI preview with approval/rejection workflow before any database write |
| 🗄️ **Auto Table Creation** | Dynamically creates PostgreSQL tables with inferred types and metadata tracking |
| 📊 **Web UI** | Drag-and-drop upload, real-time status polling, schema validation report, and IL/OTL indicators |
| 📝 **Rich Metadata** | Tracks source, URL, release date, row counts, period columns, and last available value |
| ⚡ **Async Processing** | Non-blocking background tasks via FastAPI `BackgroundTasks` |

---

## 🏗️ Architecture

```
┌─────────────┐     ┌──────────────────────────────────────────────────────────────────┐
│  Web UI /   │     │                     FastAPI Backend                              │
│  REST API   │────▶│                                                                  │
└─────────────┘     │  Upload ──▶ LLM Analysis ──▶ Preprocessing ──▶ Similarity Search│
                    │                                                        │          │
                    │              ┌─────────────────────────────────────────┤          │
                    │              ▼                                         ▼          │
                    │         Match Found?                              No Match        │
                    │              │                                         │          │
                    │    ┌─────────┴──────────┐                    ┌────────┴──────┐   │
                    │    │ Incremental Load   │                    │ One-Time Load │   │
                    │    │ (IL) Workflow      │                    │ (OTL) Workflow│   │
                    │    │                   │                    │               │   │
                    │    │ • Duplicate check │                    │ • New table   │   │
                    │    │ • Schema mapping  │                    │ • Full insert │   │
                    │    │ • Append data     │                    │               │   │
                    │    └─────────┬──────────┘                    └────────┬──────┘   │
                    │              └──────────────────┬──────────────────────┘          │
                    │                                 ▼                                 │
                    │                    Human Approval (Web UI)                        │
                    │                                 │                                 │
                    │                                 ▼                                 │
                    │                    PostgreSQL + Milvus Update                     │
                    └──────────────────────────────────────────────────────────────────┘
```

---

## 🧠 How It Works

### 1. One-Time Load (OTL)
For new datasets with no existing match:
1. File uploaded → LLM detects headers, infers column types, generates table name
2. Preprocessing: merges multi-row headers, normalizes date columns, cleans data
3. Milvus similarity search finds no match (< 85% similarity)
4. User reviews preview → Approves → Table created → Data inserted → Signature stored in Milvus

### 2. Incremental Load (IL)
For files matching an existing table (≥ 85% cosine similarity):
1. Milvus returns top similar table
2. Schema validator compares columns, maps mismatches (e.g., `baseyear` → `base_year`)
3. Duplicate detector compares period values using `dateutil` hybrid parsing
4. User sees IL preview with schema report, similarity score, and duplicate warning (if any)
5. User approves → Data appended to existing table → Metadata updated

---

## 📁 Project Structure

```
auto_sql_ingestion/
├── app/
│   ├── main.py                  # FastAPI app, endpoints, background tasks
│   ├── models.py                # Pydantic request/response models
│   ├── config.py                # Settings (env vars, thresholds)
│   └── core/
│       ├── llm_architect.py     # GPT-4o: header detection, schema inference, table naming
│       ├── preprocessor.py      # Data cleaning, header merging, date normalization
│       ├── database.py          # PostgreSQL operations (create table, insert, metadata)
│       ├── job_manager.py       # In-memory job state management
│       ├── signature_builder.py # Table signature + OpenAI embedding generation
│       ├── milvus_manager.py    # Milvus vector DB: store & search table signatures
│       ├── schema_validator.py  # Schema comparison, column mapping, duplicate detection
│       ├── incremental_loader.py# Append-only data insertion with schema alignment
│       ├── metadata_generator.py# Operational & business metadata management
│       └── logger.py            # Structured logging setup
├── static/
│   ├── index.html               # Web UI
│   ├── app.js                   # Frontend logic (polling, IL/OTL routing)
│   └── styles.css               # Styling
├── uploads/                     # Temporary uploaded files
├── processed/                   # Processed CSVs (post-approval)
├── logs/                        # Application logs
├── tests/                       # Unit & integration tests
├── requirements.txt
├── .env.example
└── README.md
```

---

## ⚙️ Installation

### Prerequisites

- Python 3.10+
- PostgreSQL 14+
- Milvus (local or cloud instance)
- OpenAI API key

### Setup

**1. Clone the repository**
```bash
git clone https://github.com/your-username/auto-sql-ingestion.git
cd auto-sql-ingestion
```

**2. Create virtual environment**
```bash
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Linux/Mac
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Configure environment variables**

Copy `.env.example` to `.env` and fill in your credentials:
```env
# OpenAI
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-4o

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=your_database
POSTGRES_USER=your_user
POSTGRES_PASSWORD=your_password

# Milvus (Vector DB)
MILVUS_HOST=localhost
MILVUS_PORT=19530
MILVUS_USER=
MILVUS_PASSWORD=
MILVUS_COLLECTION=table_signatures

# Incremental Load Settings
SIMILARITY_THRESHOLD=0.85
SIMILARITY_TOP_K=5

# App Settings
LOG_LEVEL=INFO
APPROVAL_TIMEOUT_MINUTES=30
```

**5. Start the server**
```bash
uvicorn app.main:app --reload
```

Open **http://localhost:8000** in your browser.

---

## 🖥️ Web UI

The pipeline includes a full web interface at `http://localhost:8000`:

- **Drag-and-drop** file upload (`.xlsx`, `.csv`)
- **Real-time status** polling with step-by-step progress
- **OTL Preview**: Editable table name, AI insights, sample data, metadata form
- **IL Preview**: Matched table info, similarity score, schema validation report (matching/missing/extra columns), duplicate detection warning with period ranges
- **Load type badges**: 🆕 One-Time Load vs 🔄 Incremental Load

---

## 🔌 API Reference

### Upload File
```bash
POST /upload
Content-Type: multipart/form-data

curl -X POST "http://localhost:8000/upload" \
  -F "file=@data.xlsx" \
  -F "file_description=Monthly CPI data"
```

**Response:**
```json
{
  "job_id": "79e2822c-663f-4032-9bfa-107f419fb992",
  "status": "preprocessing",
  "message": "File uploaded successfully. Preprocessing started."
}
```

---

### Check Status
```bash
GET /status/{job_id}

curl "http://localhost:8000/status/79e2822c-..."
```

**OTL Response** (`awaiting_approval`):
```json
{
  "job_id": "...",
  "status": "awaiting_approval",
  "preview": {
    "proposed_table_name": "auto_cpi_india_mth_catg",
    "columns": [{"name": "state", "type": "VARCHAR(100)"}, ...],
    "sample_rows": [...],
    "total_rows": 2157
  }
}
```

**IL Response** (`schema_mismatch` or `duplicate_data_detected`):
```json
{
  "job_id": "...",
  "status": "schema_mismatch",
  "incremental_load_preview": {
    "matched_table": {
      "table_name": "auto_cpi_state_india_mth_catg",
      "similarity_score": 0.97,
      "row_count": 23727
    },
    "validation_result": {
      "is_compatible": true,
      "match_percentage": 100.0,
      "matching_columns": ["state", "year", "month", ...],
      "missing_columns": [],
      "extra_columns": []
    },
    "new_rows_count": 2157,
    "current_rows_count": 23727,
    "total_rows_after": 25884
  },
  "duplicate_detection": {
    "status": "NEW_DATA",
    "message": "New data (2024-Jan to 2024-Dec) extends beyond existing data (up to 2023-Dec)",
    "existing_last_value": "2023-Dec",
    "new_first_value": "2024-Jan",
    "new_last_value": "2024-Dec"
  }
}
```

---

### Approve Job
```bash
POST /approve/{job_id}
Content-Type: application/x-www-form-urlencoded

curl -X POST "http://localhost:8000/approve/79e2822c-..." \
  -d "table_name=auto_cpi_india_mth_catg" \
  -d "source=Ministry of Statistics" \
  -d "source_url=https://mospi.gov.in" \
  -d "released_on=2024-01-15T00:00:00" \
  -d "updated_on=2024-01-15T00:00:00"
```

### Reject Job
```bash
POST /reject/{job_id}
```

---

## 📊 Job Status Flow

```
preprocessing
    │
    ├──▶ similarity_search
    │         │
    │    Match ≥ 85%?
    │         │
    │    YES──▶ schema_mismatch / duplicate_data_detected
    │    NO ──▶ awaiting_approval (OTL)
    │
    ▼
  approved
    │
    ├──▶ completed              (OTL success)
    └──▶ incremental_load_completed  (IL success)
         failed                 (any error)
```

---

## 🏷️ Table Naming Convention

The LLM generates standardized names following: `auto_<domain>_<geo>_<time>_<grain>`

| Component | Description | Examples |
|---|---|---|
| `domain` | Data domain | `cpi`, `iip`, `gdp`, `trade` |
| `geo` | Geographic level | `india`, `state`, `district` |
| `time` | Time granularity | `mth`, `qtr`, `yr` |
| `grain` | Data dimension | `catg`, `sctg`, `sector` |

**Examples:**
- `auto_cpi_state_india_mth_catg` — CPI Statewise Monthly Category
- `auto_iip_india_mth_sctg` — IIP India Monthly SubCategory
- `auto_gdp_india_qtr_sector` — GDP India Quarterly Sector

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **API Framework** | FastAPI + Uvicorn |
| **LLM** | OpenAI GPT-4o |
| **Embeddings** | OpenAI `text-embedding-3-small` |
| **Vector DB** | Milvus (cosine similarity) |
| **Database** | PostgreSQL + psycopg2 |
| **Data Processing** | Pandas, openpyxl |
| **Date Parsing** | python-dateutil |
| **Validation** | Pydantic v2 |
| **Frontend** | Vanilla HTML/CSS/JS |

---

## 🧪 Testing

```bash
# Schema validator tests
python test_schema_validator.py

# Milvus signature tests
python test_signature_milvus.py

# Run with debug logging
uvicorn app.main:app --reload --log-level debug
```

---

## 📋 Logging

All operations are logged to `logs/ingestion.log`:

```
2026-02-07 19:46:33 - sql_ingestion - INFO - [preprocess_file:577] - [Job 79e2...] Validation report:
2026-02-07 19:46:33 - sql_ingestion - INFO - [set_similarity_results:249] - Job 79e2... similarity results stored: 1 matches
2026-02-07 19:48:25 - sql_ingestion - INFO - [append_data:38] - Appending 2157 rows to table: auto_cpi_state_india_mth_catg
```

View logs in real-time:
```bash
Get-Content logs\ingestion.log -Wait    # Windows PowerShell
tail -f logs/ingestion.log              # Linux/Mac
```

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/xml-support`)
3. Commit your changes (`git commit -m 'Add XML file support'`)
4. Push to the branch (`git push origin feature/xml-support`)
5. Open a Pull Request

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.
