# DataPilot: Multi-Agent AI Data Analysis Assistant
## Complete Technical Interview Preparation Guide

> **Based on:** Full source code audit of the project repository  
> **Prepared for:** Technical interviews — covers architecture, implementation, code, Q&A, and pitches

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [File Structure](#2-file-structure)
3. [Functionalities — Detailed Internal Mechanics](#3-functionalities)
4. [Code Explanation with Snippets](#4-code-explanation)
5. [Function Inventory](#5-function-inventory)
6. [End-to-End Workflow](#6-end-to-end-workflow)
7. [Technical Details — APIs, DBs, GenAI, Configs](#7-technical-details)
8. [Interview Questions & Answers](#8-interview-questions--answers)
9. [Project Pitches (30s / 2min / 5min)](#9-project-pitches)
10. [Improvements & Limitations](#10-improvements--limitations)

---

## 1. Project Overview

### What It Is

**DataPilot** is a full-stack, production-ready, **multi-agent AI data analysis platform** that accepts a CSV or Excel file upload and automatically performs end-to-end data analysis — from ingestion and cleaning to statistical analysis, SQL querying, pattern detection, interactive visualizations, evidence-grounded AI insights, and a 13-section executive PDF report — all streamed in real time to the user's browser.

### Purpose

The project eliminates the need for manual data analysis by orchestrating a coordinated team of specialized AI agents through a **17-node LangGraph pipeline**, each responsible for a distinct analytical phase. The system is built for business analysts, data scientists, and anyone who needs rapid, rigorous insights from raw tabular data.

### Main Features

| Feature | Description |
|---|---|
| **File Upload** | CSV, XLSX, XLS — up to 10 MB, 500K rows, 500 columns |
| **Enterprise Data Cleaning** | 12-section automatic data sanitization (deduplication, null imputation, type normalization, range validation, encoding fixes) |
| **Data Profiling** | Column-level statistics, semantic type detection, null analysis, cardinality |
| **Quality Scoring** | 0–100 quality score with A–D grade and issue classification |
| **Statistical Analysis** | Univariate moments, Pearson correlations, group-by aggregations |
| **SQL Querying** | LLM-generated DuckDB SQL with AST safety validation and self-healing fallback |
| **Pattern Detection** | Trend regression (R², p-value), Pareto/concentration, anomaly (IQR/Z-score) |
| **Interactive Charts** | Domain-adaptive Plotly charts (bar, line, scatter, box, histogram) |
| **AI Insights** | 3–6 evidence-grounded insights following a strict 8-step reasoning chain |
| **Adversarial Critic Loop** | LLM critic audits insights against ground truth; up to 2 auto-revision cycles |
| **13-Section Report** | Executive, overview, quality, findings, distribution, correlation, trend, recommendations |
| **PDF Export** | ReportLab-generated paginated PDF with branded design |
| **Real-time SSE Streaming** | Every pipeline node emits SSE progress events to the React frontend |
| **Analysis History** | In-memory + disk cache; browsable sidebar with one-click reload |
| **PII Privacy Filter** | Redacts emails, phones, SSNs, credit cards, API keys from LLM prompts |
| **Multi-LLM Failover** | Groq (3 keys) → Gemini (2 keys) → MockProvider with 60-second cooldowns |

### Architecture

```
Frontend (React + Vite + Tailwind)
        |  HTTP POST multipart/form-data
        |  <── SSE text/event-stream (real-time progress)
        ▼
Backend (FastAPI + Uvicorn on Python 3.11)
        |
        ▼
  17-Node LangGraph StateGraph (orchestration core)
        |
  ┌─────────────────────────────────────────────┐
  │  Services Layer (deterministic computation)  │
  │  ingestion / profiling / cleaning / stats    │
  │  sql / patterns / visualization / reporting  │
  └─────────────────────────────────────────────┘
        |
  ┌─────────────────────────────────────────────┐
  │  Agents Layer (LLM-powered reasoning)        │
  │  understand_dataset / plan_analysis          │
  │  generate_sql / generate_insights            │
  │  critic_review / generate_report             │
  └─────────────────────────────────────────────┘
        |
  ┌──────────────────────┐
  │  DuckDB (in-memory)  │  ← Dataset tables registered here
  └──────────────────────┘
        |
  ┌──────────────────────┐
  │  LLM Router          │  Groq → Gemini → Mock
  └──────────────────────┘
```

### Technology Stack

**Backend:** Python 3.11, FastAPI, Uvicorn, LangGraph, DuckDB, Pandas, NumPy, SciPy, Groq SDK (LLaMA 3.3 70B), Google GenAI SDK (Gemini 2.5 Flash), ReportLab, Pydantic v2, openpyxl

**Frontend:** React 18, Vite 5, Tailwind CSS 3, Framer Motion, Plotly.js / react-plotly.js, Lucide React

**Deployment:** Render.com (YAML-configured backend + static frontend)

---

## 2. File Structure

```
multi-agent-ai-data-analysis-assistant/
├── backend/
│   ├── app/
│   │   ├── main.py                    # FastAPI app factory + CORS + exception handlers + router mounting
│   │   ├── api/
│   │   │   └── routes/
│   │   │       ├── analysis.py        # POST /api/analyze/stream — SSE pipeline trigger
│   │   │       ├── upload.py          # POST /api/upload — file ingestion
│   │   │       ├── profile.py         # GET /api/dataset/{id}/profile, /quality
│   │   │       ├── report.py          # GET report, PDF, cleaned CSV/Excel; DELETE; history
│   │   │       └── health.py          # GET /api/health
│   │   ├── agents/
│   │   │   ├── understand_dataset.py  # LLM: infers domain, KPIs, entities, business questions
│   │   │   ├── plan_analysis.py       # LLM: generates validated analysis plan
│   │   │   ├── generate_sql.py        # LLM: generates DuckDB SQL queries from plan
│   │   │   ├── generate_insights.py   # LLM: 4-part evidence-grounded insights
│   │   │   ├── critic_review.py       # LLM: adversarial 10-point fact-check audit
│   │   │   ├── revise_insights.py     # Orchestrates generate→critic→revise loop (max 2)
│   │   │   └── generate_report.py     # LLM: compiles 13-section markdown report
│   │   ├── core/
│   │   │   ├── config.py              # pydantic-settings: all env vars
│   │   │   ├── exceptions.py          # AppBaseException hierarchy
│   │   │   └── logging.py             # Structured logger
│   │   ├── llm/
│   │   │   ├── base.py                # Abstract LLMProvider interface
│   │   │   ├── groq_provider.py       # Groq API (LLaMA 3.3 70B, JSON mode)
│   │   │   ├── gemini_provider.py     # Google Gemini (2.5 Flash, native JSON schema)
│   │   │   ├── mock_provider.py       # Deterministic mock for offline/degraded mode
│   │   │   ├── router.py              # LLMRouter: failover, cooldowns, PII filter
│   │   │   └── privacy.py             # DataPrivacyFilter: regex PII redaction
│   │   ├── models/                    # 13 Pydantic v2 data models
│   │   │   ├── dataset.py, profile.py, quality.py, cleaning.py
│   │   │   ├── understanding.py, plan.py, statistics.py, sql.py
│   │   │   ├── patterns.py, visualization.py, insights.py
│   │   │   ├── critic.py, report.py
│   │   ├── orchestration/
│   │   │   ├── graph.py               # 17-node LangGraph StateGraph
│   │   │   ├── state.py               # AnalysisWorkflowState TypedDict
│   │   │   └── events.py              # STEP_METADATA, format_sse_event()
│   │   └── services/
│   │       ├── ingestion/
│   │       │   ├── validator.py        # FileValidator: filename, extension, size
│   │       │   ├── loader.py           # DatasetLoader: CSV encoding fallback, column sanitize
│   │       │   └── duckdb_manager.py   # DuckDBManager singleton
│   │       ├── profiling/profiler.py   # DatasetProfiler: per-column stats
│   │       ├── quality/checker.py      # QualityChecker: issue detection, 0-100 score
│   │       ├── cleaning/cleaner.py     # DataCleaner: 12-section pipeline (1266 lines)
│   │       ├── statistics/engine.py    # StatisticalEngine: univariate, correlation, group-by
│   │       ├── sql/
│   │       │   ├── validator.py        # SQLValidator: keyword blocklist, AST checks
│   │       │   └── executor.py         # SQLExecutor: execute, self-heal, validate intent
│   │       ├── patterns/detector.py    # PatternDetector: trend, concentration, anomaly
│   │       ├── visualization/
│   │       │   ├── chart_generator.py  # ChartGenerator: domain-adaptive Plotly JSON
│   │       │   └── role_classifier.py  # SemanticRoleClassifier: column semantic roles
│   │       └── reporting/
│   │           ├── report_builder.py   # ReportBuilder: full pipeline + cache
│   │           └── pdf_exporter.py     # PDFExporter: ReportLab branded PDF
│   ├── tests/                          # 22 pytest test files
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── App.jsx                     # Root: stage machine, SSE consumer, tab router
│       ├── services/apiConfig.js       # Dynamic API URL resolution
│       └── components/
│           ├── PipelineTracker.jsx     # Live 17-step progress tracker
│           ├── OverviewTab.jsx         # Summary cards, insights, SQL results
│           ├── DataQualityTab.jsx      # Quality score, grade, issue breakdown
│           ├── StatisticsTab.jsx       # Univariate stats, correlations
│           ├── VisualizationsTab.jsx   # Plotly chart renderer
│           ├── InsightsTab.jsx         # AI insight cards with confidence badges
│           ├── ReportMarkdownTab.jsx   # Full markdown report renderer
│           ├── CleanDataTab.jsx        # Before/after cleaning diff, downloads
│           ├── HistorySidebar.jsx      # Past analyses, reload, delete
│           └── ErrorBoundary.jsx       # React error boundary
├── samples/                            # clean_dataset.csv, messy_dataset.csv, student_scores.csv
├── render.yaml                         # Render.com deployment config
└── interview.md                        # This file
```

---

## 3. Functionalities

### 3.1 File Ingestion Pipeline

**File:** `backend/app/services/ingestion/validator.py`

`FileValidator.validate_file_metadata()` sanitizes the filename (strips path components, replaces special chars with `_`), validates extension against `{.csv, .xlsx, .xls}`, checks file size (>0 and ≤ `MAX_UPLOAD_SIZE_MB * 1024 * 1024`, default 10 MB), returns `(sanitized_name, extension)`.

**File:** `backend/app/services/ingestion/loader.py`

`DatasetLoader.load_and_sanitize()` tries 5 encodings (`utf-8`, `utf-8-sig`, `latin-1`, `cp1252`, `iso-8859-1`) and 4 delimiters (`,`, `;`, `\t`, `|`) for CSV. Uses `openpyxl` for `.xlsx`. Enforces limits (500K rows, 500 columns). Sanitizes column names (removes non-alphanumeric, prevents digit-prefix, ensures uniqueness). Returns `(pd.DataFrame, List[ColumnSchema])`.

**File:** `backend/app/services/ingestion/duckdb_manager.py`

`DuckDBManager.register_dataframe()` is a singleton with a single in-memory DuckDB connection. Registers DataFrame as: `CREATE OR REPLACE TABLE {name} AS SELECT * FROM temp_df_view`. Maintains `registered_tables: Dict[str, str]` (dataset_id → table_name).

---

### 3.2 Dataset Profiling

**File:** `backend/app/services/profiling/profiler.py`

`DatasetProfiler.profile_dataset()` iterates every column, infers `semantic_type` (numeric/categorical/datetime/boolean/identifier/text), computes null count, unique count, sample values, numeric stats (min/max/mean/median/std/q25/q75), categorical top-N frequencies, datetime ranges. Detects dataset-level duplicate rows. Returns `DatasetProfile`.

---

### 3.3 Data Quality Auditing

**File:** `backend/app/services/quality/checker.py`

`QualityChecker.audit_dataset()` checks for: missing values, duplicates, low-variance columns, potential outliers, inconsistent label casing, mixed types. Classifies each issue as `confirmed/suspicious/informational`. Computes 0–100 quality score with grade A (≥90)/B (≥75)/C (≥60)/D (<60).

---

### 3.4 Enterprise Data Cleaning — 12 Sections

**File:** `backend/app/services/cleaning/cleaner.py` (1,266 lines)

`DataCleaner.clean_dataset()` applies 12 sequential, deterministic passes:

| Section | Operation |
|---|---|
| 1 | **Structural Sanitization** — remove all-NaN rows/cols, reset index |
| 2 | **Null Placeholder Purge** — 30+ string null variants (`"n/a"`, `"null"`, `"--"`, `"#N/A"`, `"invalid_date"`) → `np.nan` |
| 3 | **Mojibake & Encoding Repair** — fixes garbled chars (`Ã©` → `é`) via `MOJIBAKE_MAP` |
| 4 | **Whitespace & Case Normalization** — strip whitespace, title-case names |
| 5 | **Unit Stripping** — removes `$`, `€`, `kg`, `%` from numeric-like strings |
| 6 | **Categorical Normalization** — maps abbrevs to canonical forms (`"n"` → `"North"`, `"mob"` → `"Mobile"`, `"M"` → `"Male"`) |
| 7 | **Boolean Standardization** — `"yes"/"y"/"1"/"true"/"active"` → `True` |
| 8 | **Date Normalization** — multi-format parsing via `dateutil.parser`, ISO output |
| 9 | **Numeric Type Coercion** — string-numeric values → float/int |
| 10 | **Out-of-Range Detection** — IQR-based outlier capping at 1.5× boundaries |
| 11 | **Typo Collapsing** — `difflib.get_close_matches()` unifies similar labels |
| 12 | **Missing Value Imputation** — median (numeric), mode (categorical); full `CleaningSummary` audit log |

Returns `(cleaned_df, CleaningSummary)` with before/after counts, change log, nulls imputed, duplicates removed.

---

### 3.5 LangGraph Orchestration — 17-Node Pipeline

**File:** `backend/app/orchestration/graph.py`

`build_analysis_graph()` creates a `StateGraph(AnalysisWorkflowState)`. Each node is a pure Python function. A **conditional edge** from `revise_insights` enables the critic feedback loop.

```
START
  → validate_file          [1]  FileValidator
  → load_dataset           [2]  DatasetLoader + DuckDBManager
  → profile_and_audit      [3]  DatasetProfiler + QualityChecker
  → clean_and_standardize  [4]  DataCleaner (12-section)
  → understand_dataset     [5]  LLM: DatasetUnderstandingAgent
  → plan_analysis          [6]  LLM: AnalysisPlanningAgent
  → run_statistical_analysis [7] StatisticalEngine
  → generate_sql           [8]  LLM: SQLGenerationAgent
  → validate_sql           [9]  SQLValidator (AST security)
  → execute_sql            [10] SQLExecutor (DuckDB + self-heal)
  → detect_patterns        [11] PatternDetector
  → render_charts          [12] ChartGenerator (Plotly JSON)
  → generate_insights      [13] LLM: InsightGenerationAgent
  → critic_review          [14] LLM: CriticReviewAgent (temp=0.0)
  → revise_insights        [15] Conditional routing node
       ├── if rejected AND revisions < 2 → back to generate_insights
       └── if approved OR max revisions → generate_report
  → generate_report        [16] LLM: ReportGenerationAgent (13 sections)
  → render_pdf             [17] PDFExporter + ReportBuilder cache
  → END
```

`AnalysisWorkflowState` is a `TypedDict` (shared blackboard). Each node reads what it needs and writes only its outputs.

---

### 3.6 LLM Agent System

All agents use `llm_router.complete(agent_name, messages, response_model, temperature)` and receive a validated Pydantic model.

**`DatasetUnderstandingAgent`** — Infers domain, narrative summary, primary entity, candidate KPIs, segmentation dimensions, 3–5 business questions, data caveats. Returns `DatasetUnderstanding`.

**`AnalysisPlanningAgent`** — Generates analytical goal, numeric columns for statistics, correlation pairs, 2–4 group-by plans, 2–4 SQL goals, chart recommendations. **Post-validates every column reference** against actual profile via `validate_and_clean_plan()`. Returns `AnalysisPlan`.

**`SQLGenerationAgent`** — Provides 9-rule system prompt (no DDL, no mutation, double-quote identifiers, NULLIF for division, no semicolons, only valid column names). Returns `List[GeneratedSQLQuery]`.

**`InsightGenerationAgent`** — Forces 8-step reasoning chain: `Data → Analysis → Finding → Evidence → Interpretation → Business Impact → Recommendation → Confidence`. Includes 9 statistical guardrails (no causation from correlation, proper skewness, no false Pareto). On revision: injects critic's structured critique into user prompt. Returns `InsightCollection` (3–6 `InsightItem` objects).

**`CriticReviewAgent`** — Runs at `temperature=0.0` (deterministic). Audits insights against a 10-point checklist. Returns `CriticReviewResult` with `approved: bool`, `unsupported_claims`, `required_corrections`.

**`InsightRevisionOrchestrator`** — `MAX_REVISIONS = 2`. If max exceeded and still unapproved: downgrades flagged insights' confidence to `"Caveat"` and appends a caveat to the finding text.

**`ReportGenerationAgent`** — Compiles all pipeline outputs into a structured prompt, returns `AnalysisReport` with 13 `ReportSection` objects.

---

### 3.7 LLM Router & Multi-Provider Failover

**File:** `backend/app/llm/router.py`

Priority: Groq key 1 → Groq key 2 → Groq key 3 → Gemini key 1 → Gemini key 2 → MockLLMProvider.

**Cooldown:** If a provider raises `LLMProviderError` matching `"rate"`, `"429"`, `"quota"`, or `"limit"`, `mark_cooldown()` stores `time.time() + 60.0` for that key. Subsequent calls skip cooled keys.

**PII Redaction:** `DataPrivacyFilter.redact_text()` runs on all message content before sending to LLMs. Redacts: emails, phone numbers, SSNs, credit cards, API key patterns.

**Gemini:** Uses `response_mime_type="application/json"` + `response_schema=response_model` in `GenerateContentConfig` for native schema-constrained output.

**Groq:** Injects full JSON Schema of Pydantic model into system prompt, uses `response_format={"type": "json_object"}`.

Both parse with `response_model.model_validate_json(raw_text)`.

---

### 3.8 SQL Safety Pipeline

**File:** `backend/app/services/sql/validator.py`

`SQLValidator.validate_sql()` — 6 layers:
1. Strip trailing `;`; reject multi-statement (`;` inside)
2. Must start with `SELECT` or `WITH`
3. Keyword blocklist: 20+ DDL/DML keywords (`DROP`, `DELETE`, `INSERT`, `ALTER`, `COPY`, `ATTACH`, `PRAGMA`, `EXEC`, `GRANT`, etc.)
4. Function blocklist: `read_csv`, `read_parquet`, `httpfs`, `glob`, `write_csv`, etc.
5. URL/protocol detection: `https://`, `s3://`, `gcs://`, `file://`
6. Table name verification + auto-correction of generic `FROM dataset` → actual table name

**File:** `backend/app/services/sql/executor.py`

`SQLExecutor.execute_single_query()`: validate → inspect schema → repair syntax → execute → validate intent vs result.

`repair_query_syntax()`: fixes table name used as column in SELECT/GROUP BY, replaces `SUM`/`AVG` on string columns with `COUNT(DISTINCT ...)`.

`synthesize_fallback_query()`: If execution fails, synthesizes a guaranteed-valid query from schema + query intent keywords.

`validate_query_intent_against_result()`: Detects mismatch — e.g. query named `"unrestocked_products"` but returning only aggregates.

---

### 3.9 Pattern Detection

**File:** `backend/app/services/patterns/detector.py`

- **Trends:** Linear regression via `scipy.stats.linregress`; computes R², p-value, direction, statistical significance
- **Concentrations:** Share percentages of top categories; Pareto label only if top ~20% holds ≥75–80% of volume
- **Anomalies:** Z-score (>3σ) and IQR (1.5×) methods; severity classified as `high/medium/low`
- **Seasonality:** Checks datetime columns for monthly/quarterly periodic patterns

---

### 3.10 Visualization — Domain-Adaptive Charts

**File:** `backend/app/services/visualization/role_classifier.py`

`SemanticRoleClassifier.classify_column()` assigns roles: `identifier | datetime | measure_additive | measure_rate | category_low | category_medium | category_high | binary | text`. Uses name regex, dtype, unique count ratio, semantic_type from profile.

**File:** `backend/app/services/visualization/chart_generator.py` (32 KB)

`ChartGenerator.generate_all()` uses column roles to select appropriate chart types. Generates 8–12 Plotly chart specs as JSON. Returns `ChartCollection` with `List[ChartSpec]`.

---

### 3.11 Real-Time SSE Streaming

**File:** `backend/app/api/routes/analysis.py`

`event_generator_from_graph()` uses `analysis_workflow_graph.astream(initial_state)`. For each node, emits `step_complete` SSE event with `step_index`, `label`, `agent`, and a `preview` snippet. `asyncio.sleep(0.05)` between nodes for smooth rendering.

SSE format:
```
event: step_complete
data: {"step": "profile_and_audit", "step_index": 3, "total_steps": 17, "preview": {...}}
```

**Frontend** (`App.jsx`): uses `fetch()` + `response.body.getReader()` (not `EventSource`, which doesn't support POST). Buffers partial SSE chunks with `split('\n\n')`, keeps incomplete chunk in buffer, parses `event:` and `data:` lines.

---

### 3.12 Report & PDF Export

**File:** `backend/app/services/reporting/report_builder.py`

`ReportBuilder`: two-tier cache (in-memory dict + disk `.cache/reports/*.json` and `.cache/cleaned/*.csv`). `list_history()` reads all cached reports. `delete_report()` removes from memory, disk, and DuckDB. `build_report_from_dataset()` is a full re-run pipeline for on-demand report regeneration.

**File:** `backend/app/services/reporting/pdf_exporter.py`

`PDFExporter.generate_pdf()` uses `reportlab.platypus`. Custom `NumberedCanvas` implements two-pass rendering for accurate `"Page X of N"` footers. Branded color palette: `#3E2723` (dark), `#AD8B73` (primary), `#CEAB93` (accent).

---

## 4. Code Explanation

### 4.1 `AnalysisWorkflowState` — The Pipeline's Shared Blackboard

**File:** `backend/app/orchestration/state.py`

```python
class AnalysisWorkflowState(TypedDict, total=False):
    # Input
    dataset_id: str
    filename: str
    file_bytes: Optional[bytes]
    file_extension: str
    table_name: str         # switches from raw to cleaned at Node 4
    raw_table_name: str
    cleaned_table_name: str
    df: Optional[pd.DataFrame]    # active DataFrame
    raw_df: Optional[pd.DataFrame]
    cleaned_df: Optional[pd.DataFrame]
    cleaning_summary: Optional[Dict[str, Any]]

    # Pipeline Artifacts
    profile: Optional[DatasetProfile]
    quality: Optional[QualityReport]
    understanding: Optional[DatasetUnderstanding]
    plan: Optional[AnalysisPlan]
    statistics: Optional[StatisticalAnalysisResult]
    sql_queries: List[GeneratedSQLQuery]
    validated_sql_queries: List[GeneratedSQLQuery]
    sql_results: Optional[SQLAnalysisResult]
    patterns: Optional[PatternDetectionResult]
    charts: Optional[ChartCollection]
    insights: Optional[InsightCollection]
    critic_review: Optional[CriticReviewResult]
    critic_approved: bool
    revision_count: int           # 0, 1, or 2
    revision_critique: Optional[str]
    report: Optional[AnalysisReport]
    pdf_bytes: Optional[bytes]

    # Execution Metadata
    current_step: str
    step_index: int
    total_steps: int
    status_label: str
```

`total=False` means no field is mandatory at construction — nodes add outputs incrementally.

---

### 4.2 The Conditional Edge — Critic Revision Loop

**File:** `backend/app/orchestration/graph.py`

```python
# Conditional edge from revise_insights
graph.add_conditional_edges(
    "revise_insights",
    should_continue_revision,
    {
        "generate_insights": "generate_insights",  # loop back
        "generate_report": "generate_report"       # proceed
    }
)

def should_continue_revision(state) -> Literal["generate_insights", "generate_report"]:
    """Loop back to generate_insights if unapproved and revision < 2, else proceed."""
    if not state.get("critic_approved", True) and state.get("revision_count", 0) <= 2:
        return "generate_insights"
    return "generate_report"
```

---

### 4.3 `LLMRouter.complete()` — Provider Failover with PII Redaction

**File:** `backend/app/llm/router.py`

```python
def complete(self, agent_name, messages, response_model, temperature=0.1) -> T:
    # Step 1: PII redaction on ALL messages before any external call
    sanitized_messages = [
        {"role": m["role"], "content": DataPrivacyFilter.redact_text(m["content"])}
        for m in messages
    ]

    # Step 2: Get available providers (skip cooled-down keys)
    available_providers = self._get_available_providers()

    # Step 3: Fallback to mock if no keys configured
    if not available_providers:
        return self.mock_provider.generate_structured(...)

    # Step 4: Try providers in priority order
    for key_id, provider in available_providers:
        try:
            return provider.generate_structured(
                messages=sanitized_messages,
                response_model=response_model,
                temperature=temperature
            )
        except LLMProviderError as pe:
            if "rate" in str(pe).lower() or "429" in str(pe):
                self.mark_cooldown(key_id)   # 60-second cooldown
            continue

    # Step 5: Graceful final fallback to mock
    return self.mock_provider.generate_structured(...)
```

---

### 4.4 Anti-Hallucination Post-Validation in Plan Agent

**File:** `backend/app/agents/plan_analysis.py`

```python
@classmethod
def validate_and_clean_plan(cls, plan, valid_columns, numeric_columns):
    valid_cols_lower = {c.lower(): c for c in valid_columns}
    num_cols_lower = {c.lower(): c for c in numeric_columns}

    # Remove LLM-hallucinated numeric columns
    plan.descriptive_numeric_columns = [
        num_cols_lower[c.lower()]
        for c in plan.descriptive_numeric_columns
        if c.lower() in num_cols_lower
    ] or list(numeric_columns)[:4]  # fallback to first 4 real columns

    # Remove invalid correlation pairs (both must be real numerics)
    plan.correlation_pairs = [
        pair for pair in plan.correlation_pairs
        if pair[0].lower() in num_cols_lower and pair[1].lower() in num_cols_lower
    ]
    # ... similarly for group_by_analyses and sql_query_goals
    return plan
```

This guarantees **zero hallucinated column names** reach the statistical or SQL pipeline.

---

### 4.5 SQL Self-Healing Execution

**File:** `backend/app/services/sql/executor.py`

```python
@classmethod
def execute_single_query(cls, query, table_name):
    # 1. Security validation
    is_safe, reason, sanitized_sql = SQLValidator.validate_sql(query.sql, expected_table=table_name)
    if not is_safe:
        return SQLExecutionResult(execution_status="rejected", ...)

    # 2. Schema inspection
    num_cols, cat_cols = cls.get_table_schema(table_name)

    # 3. Proactive syntax repair
    query_to_run = cls.repair_query_syntax(sanitized_sql, table_name, num_cols, cat_cols)

    # 4. Execute primary query
    try:
        result_df = duckdb_manager.conn.execute(query_to_run).df()
        return SQLExecutionResult(execution_status="success", ...)
    except Exception:
        # 5. SELF-HEALING: synthesize guaranteed-valid fallback
        fallback_sql = cls.synthesize_fallback_query(
            query.name, query.purpose, table_name, num_cols, cat_cols
        )
        result_df = duckdb_manager.conn.execute(fallback_sql).df()
        return SQLExecutionResult(execution_status="success", ...)
```

---

### 4.6 `DataPrivacyFilter.redact_text()` — PII Protection

**File:** `backend/app/llm/privacy.py`

```python
class DataPrivacyFilter:
    EMAIL_REGEX = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}", re.IGNORECASE)
    PHONE_REGEX = re.compile(r"\b(?:\+?\d{1,3}[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b")
    SSN_REGEX = re.compile(r"\b\d{3}-\d{2}-\d{4}\b")
    CREDIT_CARD_REGEX = re.compile(r"\b(?:\d{4}[-\s]?){3}\d{4}\b")
    API_KEY_REGEX = re.compile(
        r"\b(?:api[_-]?key|secret|token|bearer|password)[=:\s]+[A-Za-z0-9_\-\.]{8,}\b",
        re.IGNORECASE
    )

    @classmethod
    def redact_text(cls, text: str) -> str:
        redacted = cls.EMAIL_REGEX.sub("[EMAIL_REDACTED]", text)
        redacted = cls.PHONE_REGEX.sub("[PHONE_REDACTED]", redacted)
        redacted = cls.SSN_REGEX.sub("[SSN_REDACTED]", redacted)
        redacted = cls.CREDIT_CARD_REGEX.sub("[CARD_REDACTED]", redacted)
        redacted = cls.API_KEY_REGEX.sub("[SECRET_REDACTED]", redacted)
        return redacted
```

---

### 4.7 `format_sse_event()` and `clean_nan_and_inf()`

**File:** `backend/app/orchestration/events.py`

```python
def clean_nan_and_inf(obj: Any) -> Any:
    """Recursively replaces NaN/Inf with None for JSON compatibility."""
    if isinstance(obj, float):
        if math.isnan(obj) or math.isinf(obj):
            return None
        return obj
    elif isinstance(obj, dict):
        return {k: clean_nan_and_inf(v) for k, v in obj.items()}
    elif isinstance(obj, list):
        return [clean_nan_and_inf(v) for v in obj]
    return obj

def format_sse_event(event_type: str, data: Dict[str, Any]) -> str:
    sanitized_data = clean_nan_and_inf(data)
    payload = {**sanitized_data, "timestamp": datetime.now(timezone.utc).isoformat()}
    return f"event: {event_type}\ndata: {json.dumps(payload, default=str)}\n\n"
```

`float('nan')` and `float('inf')` are invalid JSON. Statistical computations frequently produce them. Without `clean_nan_and_inf()`, the frontend `JSON.parse()` throws `SyntaxError`.

---

### 4.8 Frontend SSE Parsing — Manual Chunked Reading

**File:** `frontend/src/App.jsx`

```javascript
// fetch() used instead of EventSource because SSE doesn't support POST
const response = await fetch(`${apiBase}/api/analyze/stream`, {
    method: 'POST',
    body: formData,
    signal: abortControllerRef.current.signal
});

const reader = response.body.getReader();
const decoder = new TextDecoder('utf-8');
let buffer = '';

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n\n');
    buffer = lines.pop();  // keep incomplete last chunk in buffer

    for (const block of lines) {
        let eventType = 'message', rawData = '';
        for (const line of block.split('\n')) {
            if (line.startsWith('event:')) eventType = line.replace('event:', '').trim();
            else if (line.startsWith('data:')) rawData = line.replace('data:', '').trim();
        }
        const eventPayload = JSON.parse(rawData);
        // dispatch to React state based on eventType
    }
}
```

---

### 4.9 API Base URL Resolution

**File:** `frontend/src/services/apiConfig.js`

```javascript
export const getApiBaseUrl = () => {
    // 1. User-saved override in localStorage
    const savedUrl = localStorage.getItem('datapilot_api_base_url');
    if (savedUrl) return savedUrl.trim().replace(/\/+$/, '');

    // 2. Vite build-time env var (only if non-localhost)
    const envUrl = import.meta.env.VITE_API_BASE_URL;
    if (envUrl && !envUrl.includes('localhost')) return envUrl.trim();

    // 3. Intelligent Render.com hostname inference
    const hostname = window.location.hostname;
    if (hostname.includes('-frontend.onrender.com')) {
        return `https://${hostname.replace('-frontend.', '-backend.')}`;
    }

    // 4. localhost default
    return 'http://localhost:8000';
};
```

---

## 5. Function Inventory

### Agent Classes

| Class | File | Key Method | Purpose |
|---|---|---|---|
| `DatasetUnderstandingAgent` | `agents/understand_dataset.py` | `.analyze(profile, quality, filename)` | Domain/KPI/question inference |
| `AnalysisPlanningAgent` | `agents/plan_analysis.py` | `.plan(profile, understanding)` | Generate + validate analysis plan |
| `AnalysisPlanningAgent` | `agents/plan_analysis.py` | `.validate_and_clean_plan(plan, valid_cols, num_cols)` | Remove hallucinated columns |
| `SQLGenerationAgent` | `agents/generate_sql.py` | `.generate(profile, plan, table_name)` | Generate safe DuckDB SQL queries |
| `InsightGenerationAgent` | `agents/generate_insights.py` | `.generate(..., revision_critique=None)` | 3–6 evidence-grounded insights |
| `CriticReviewAgent` | `agents/critic_review.py` | `.review(insights, statistics, sql_results, patterns)` | Adversarial 10-point fact-check |
| `InsightRevisionOrchestrator` | `agents/revise_insights.py` | `.run_generation_and_critic_loop(...)` | Generate→critic→revise (max 2) |
| `ReportGenerationAgent` | `agents/generate_report.py` | `.generate(...)` | 13-section executive report |

### Service Classes

| Class | File | Key Method | Purpose |
|---|---|---|---|
| `FileValidator` | `services/ingestion/validator.py` | `.validate_file_metadata(filename, size, content_type)` | Sanitize, check extension & size |
| `DatasetLoader` | `services/ingestion/loader.py` | `.load_and_sanitize(content, extension)` | Multi-encoding CSV + column sanitization |
| `DuckDBManager` | `services/ingestion/duckdb_manager.py` | `.register_dataframe(df, dataset_id, table_name)` | Register Pandas DF as DuckDB table |
| `DatasetProfiler` | `services/profiling/profiler.py` | `.profile_dataset(df, dataset_id, table_name)` | Full per-column statistical profile |
| `QualityChecker` | `services/quality/checker.py` | `.audit_dataset(df, profile)` | Issue detection + 0–100 score |
| `DataCleaner` | `services/cleaning/cleaner.py` | `.clean_dataset(df, dataset_id, filename)` | 12-section enterprise cleaning |
| `DataCleaner` | `services/cleaning/cleaner.py` | `.export_csv_bytes(df)` / `.export_excel_bytes(df, sheet_name)` | Export cleaned data |
| `StatisticalEngine` | `services/statistics/engine.py` | `.run_analysis(df, profile, plan)` | Univariate + correlations + group-by |
| `SQLValidator` | `services/sql/validator.py` | `.validate_sql(sql, expected_table, valid_columns)` | 6-layer SQL security check |
| `SQLExecutor` | `services/sql/executor.py` | `.execute_queries(queries, dataset_id, table_name)` | Batch query execution with recovery |
| `SQLExecutor` | `services/sql/executor.py` | `.repair_query_syntax(sql, table, num_cols, cat_cols)` | Fix common LLM SQL mistakes |
| `SQLExecutor` | `services/sql/executor.py` | `.synthesize_fallback_query(name, purpose, table, ...)` | Guaranteed-valid fallback SQL |
| `PatternDetector` | `services/patterns/detector.py` | `.detect_all(df, profile)` | Trends + concentrations + anomalies |
| `SemanticRoleClassifier` | `services/visualization/role_classifier.py` | `.classify_column(col_prof, total_rows, series)` | Semantic role per column |
| `ChartGenerator` | `services/visualization/chart_generator.py` | `.generate_all(df, profile, plan, user_query)` | Domain-adaptive Plotly chart collection |
| `ReportBuilder` | `services/reporting/report_builder.py` | `.build_report_from_dataset(df, dataset_id, ...)` | Full pipeline re-run for report |
| `ReportBuilder` | `services/reporting/report_builder.py` | `.cache_report(report)` / `.get_report(id)` | Two-tier memory + disk caching |
| `PDFExporter` | `services/reporting/pdf_exporter.py` | `.generate_pdf(report)` | ReportLab branded PDF bytes |

### LLM Layer

| Class | File | Key Method | Purpose |
|---|---|---|---|
| `LLMProvider` | `llm/base.py` | `.generate_structured(messages, response_model, temperature)` | Abstract interface |
| `GroqProvider` | `llm/groq_provider.py` | `.generate_structured(...)` | Groq API with JSON mode |
| `GeminiProvider` | `llm/gemini_provider.py` | `.generate_structured(...)` | Gemini with native schema |
| `MockLLMProvider` | `llm/mock_provider.py` | `.generate_structured(...)` | Deterministic offline fallback |
| `LLMRouter` | `llm/router.py` | `.complete(agent_name, messages, response_model, temperature)` | Multi-provider failover + PII |
| `LLMRouter` | `llm/router.py` | `.mark_cooldown(key_id)` | 60s rate-limit cooldown |
| `DataPrivacyFilter` | `llm/privacy.py` | `.redact_text(text)` | Regex PII redaction |

### LangGraph Node Functions (all in `orchestration/graph.py`)

| Node Function | Step | Key State Writes |
|---|---|---|
| `validate_file_node` | 1 | `filename`, `file_extension` |
| `load_dataset_node` | 2 | `dataset_id`, `table_name`, `raw_df`, `df` |
| `profile_and_audit_node` | 3 | `profile`, `quality` |
| `clean_and_standardize_node` | 4 | `df` (cleaned), `cleaned_df`, `cleaning_summary`, updated `profile`, `table_name` |
| `understand_dataset_node` | 5 | `understanding` |
| `plan_analysis_node` | 6 | `plan` |
| `run_statistical_analysis_node` | 7 | `statistics` |
| `generate_sql_node` | 8 | `sql_queries` |
| `validate_sql_node` | 9 | `validated_sql_queries` |
| `execute_sql_node` | 10 | `sql_results` |
| `detect_patterns_node` | 11 | `patterns` |
| `render_charts_node` | 12 | `charts` |
| `generate_insights_node` | 13 | `insights` |
| `critic_review_node` | 14 | `critic_review`, `critic_approved` |
| `revise_insights_node` | 15 | `revision_count`, `revision_critique`, `critic_approved` |
| `generate_report_node` | 16 | `report` |
| `render_pdf_node` | 17 | `pdf_bytes` |

### API Routes

| Method | Path | File | Description |
|---|---|---|---|
| GET | `/api/health` | `health.py` | Health check |
| POST | `/api/upload` | `upload.py` | Upload + ingest file |
| GET | `/api/dataset/{id}/profile` | `profile.py` | Column profile |
| GET | `/api/dataset/{id}/quality` | `profile.py` | Quality report |
| POST | `/api/analyze/stream` | `analysis.py` | SSE: full 17-step pipeline |
| GET | `/api/dataset/{id}/report` | `report.py` | Full analysis report JSON |
| GET | `/api/dataset/{id}/report/pdf` | `report.py` | Download PDF report |
| GET | `/api/dataset/{id}/download/cleaned-csv` | `report.py` | Download cleaned CSV |
| GET | `/api/dataset/{id}/download/cleaned-excel` | `report.py` | Download cleaned XLSX |
| GET | `/api/dataset/{id}/cleaned-preview` | `report.py` | Top 20 rows preview |
| GET | `/api/dataset/history` | `report.py` | List analysis history |
| DELETE | `/api/dataset/history` | `report.py` | Clear all history |
| DELETE | `/api/dataset/{id}` | `report.py` | Delete single analysis |

---

## 6. End-to-End Workflow

### Full User Journey

```
User Action                    Frontend Event                  Backend Event
──────────────────────────────────────────────────────────────────────────────
1. Open app                    stage='upload'
2. Select file / sample        setSelectedFile(file)
3. Click "Analyze"             handleStartAnalysis()
   → FormData with file        POST /api/analyze/stream ──────► FastAPI
4.                             SSE StreamingResponse ◄──────────
   → response.body.getReader() begins SSE consumption

   PIPELINE EVENTS (emitted per node completion):

   SSE: pipeline_start ────────────────────────► setStage('streaming')
   SSE: step_complete(validate_file, 1/17)
   SSE: step_complete(load_dataset, 2/17)
   SSE: step_complete(profile_and_audit, 3/17)  ► preview: {quality_score, grade}
   SSE: step_complete(clean_and_standardize, 4/17)
   SSE: step_complete(understand_dataset, 5/17) ► preview: {domain, kpis}
   SSE: step_complete(plan_analysis, 6/17)
   SSE: step_complete(run_statistical_analysis, 7/17) ► {metrics_count}
   SSE: step_complete(generate_sql, 8/17)
   SSE: step_complete(validate_sql, 9/17)
   SSE: step_complete(execute_sql, 10/17)
   SSE: step_complete(detect_patterns, 11/17)
   SSE: step_complete(render_charts, 12/17)
   SSE: step_complete(generate_insights, 13/17) ► {insights_count, top_title}
   SSE: step_complete(critic_review, 14/17)
   [OPTIONAL: steps 13-15 repeat up to 2× if critic rejects]
   SSE: step_complete(revise_insights, 15/17)
   SSE: step_complete(generate_report, 16/17) ► {report_title, sections_count}
   SSE: step_complete(render_pdf, 17/17)
   SSE: complete ─────────────────────────────► setFinalReport(report)
                                                 setStage('dashboard')

5. Dashboard renders            activeTab='overview'
   - OverviewTab: domain, KPIs, insights, SQL results
   - DataQualityTab: score, grade, issue list
   - StatisticsTab: univariate table, correlation matrix
   - VisualizationsTab: Plotly charts
   - InsightsTab: 3–6 insight cards with confidence badges
   - ReportMarkdownTab: full 13-section markdown
   - CleanDataTab: before/after diff, download buttons

6. Download PDF                GET /api/dataset/{id}/report/pdf ──► PDF bytes
7. Download CSV                GET /api/dataset/{id}/download/cleaned-csv
8. History sidebar             HistorySidebar opens; click past entry → reload
```

### Data State Transitions

```
file_bytes (raw binary)
  → [load_dataset]          → raw_df (pd.DataFrame, raw)
  → [profile_and_audit]     → profile (DatasetProfile) + quality (QualityReport)
  → [clean_and_standardize] → cleaned_df + cleaning_summary
        (df in state becomes cleaned_df; table_name switches to dataset_{uuid})
  → [understand_dataset]    → understanding (domain, KPIs, questions)
  → [plan_analysis]         → plan (validated analytical recipe)
  → [run_statistical_analysis] → statistics (moments, correlations, group-by)
  → [generate_sql]          → sql_queries (LLM-generated DuckDB SQL)
  → [validate_sql]          → validated_sql_queries (security-checked)
  → [execute_sql]           → sql_results (actual query result rows)
  → [detect_patterns]       → patterns (trends, concentrations, anomalies)
  → [render_charts]         → charts (Plotly JSON specs)
  → [generate_insights]     → insights (InsightCollection)
  → [critic_review]         → critic_review + critic_approved
  → [revise_insights]       → (possibly loops to generate_insights)
  → [generate_report]       → report (AnalysisReport with 13 sections)
  → [render_pdf]            → pdf_bytes + cached report
```

---

## 7. Technical Details

### AI / GenAI Components

| Component | Model | Role |
|---|---|---|
| `DatasetUnderstandingAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | Domain inference, KPI identification |
| `AnalysisPlanningAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | Analytical plan generation |
| `SQLGenerationAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | DuckDB SQL synthesis |
| `InsightGenerationAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | Evidence-grounded insight writing |
| `CriticReviewAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | Adversarial fact-checking (temp=0.0) |
| `ReportGenerationAgent` | Groq LLaMA 3.3 70B / Gemini 2.5 Flash | 13-section report writing |
| `MockLLMProvider` | Pure Python | Offline/degraded-mode deterministic fallback |

**Structured Output:**
- **Groq:** Full JSON Schema injected into system prompt + `response_format={"type": "json_object"}` → `model_validate_json()`
- **Gemini:** `response_mime_type="application/json"` + `response_schema=response_model` in `GenerateContentConfig` → native schema-constrained output

**LangGraph:** `StateGraph` with `astream()` async streaming. Nodes run sequentially; conditional edge enables the revision loop.

---

### Database — DuckDB

- **Type:** In-memory columnar OLAP SQL database (no files, no server)
- **Why DuckDB:** Native Pandas integration, in-process, blazing fast for aggregations, full SQL including CTEs, no external dependency
- **Singleton:** One shared `duckdb.connect(database=":memory:")` for the whole application
- **Table naming:** `dataset_{uuid_with_underscores}`; raw tables: `raw_dataset_{uuid}`
- **Registration:** `CREATE OR REPLACE TABLE {name} AS SELECT * FROM temp_df_view`
- **Raw vs clean:** Node 2 creates `raw_{table}`, Node 4 creates `{table}` (cleaned). Downstream always uses cleaned table.

---

### APIs & External Integrations

**Groq API** — SDK `groq>=0.5.0`, model `llama-3.3-70b-versatile`, up to 3 API keys, JSON mode

**Google Gemini API** — SDK `google-genai>=0.1.1`, model `gemini-2.5-flash`, up to 2 API keys, native schema output

---

### Configuration (`core/config.py`)

```python
class Settings(BaseSettings):
    APP_NAME: str = "DataPilot:Multi-Agent Data Analyst API"
    APP_VERSION: str = "1.0.0"
    MAX_UPLOAD_SIZE_MB: int = 10
    MAX_DATASET_ROWS: int = 500000
    MAX_DATASET_COLUMNS: int = 500
    GROQ_API_KEY_1: str = ""
    GROQ_API_KEY_2: str = ""
    GROQ_API_KEY_3: str = ""
    GROQ_MODEL: str = "llama-3.3-70b-versatile"
    GEMINI_API_KEY_1: str = ""
    GEMINI_API_KEY_2: str = ""
    GEMINI_MODEL: str = "gemini-2.5-flash"
    CORS_ORIGINS: Union[List[str], str] = ["http://localhost:5173", ...]
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
```

---

### Deployment (`render.yaml`)

```yaml
services:
  - type: web
    name: datapilot-backend
    runtime: python
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn app.main:app --host 0.0.0.0 --port $PORT
    healthCheckPath: /api/health
    envVars:
      - key: CORS_ORIGINS
        value: "*"
      - key: GROQ_API_KEY_1
        sync: false   # Set via Render dashboard, not stored in YAML

  - type: web
    name: datapilot-frontend
    runtime: static
    buildCommand: npm install && npm run build
    staticPublishPath: dist
    routes:
      - type: rewrite
        source: /*
        destination: /index.html  # SPA routing support
```

---

### Exception Hierarchy

```python
AppBaseException              # Base: .message + .details
  ├── FileValidationError     # Invalid file type, size, encoding
  ├── IngestionError          # DuckDB registration/parsing failure
  ├── LLMProviderError        # LLM API call failure
  └── SQLSecurityError        # SQL validation rejection
```

Global handlers in `main.py`:
- `AppBaseException` → HTTP 400 with `{error, message, details}`
- `Exception` (unhandled) → HTTP 500 with generic message (no stack trace leaked)

---

## 8. Interview Questions & Answers

### 🟢 Basic Level

**Q1: What is this project and what problem does it solve?**

> **A:** DataPilot is a multi-agent AI-powered data analysis platform that automates the entire analytical workflow for CSV/Excel data. Business analysts and data teams spend enormous time on repetitive tasks: cleaning messy data, writing SQL queries, computing statistics, generating charts, and writing reports. DataPilot orchestrates a team of specialized AI agents through a 17-step pipeline to do all of this automatically and deliver a publication-ready 13-section report with interactive charts and evidence-grounded insights — all in real time via SSE streaming.

---

**Q2: What is LangGraph and how is it used here?**

> **A:** LangGraph is a library from LangChain for building stateful, graph-based agent workflows. It's used to build a `StateGraph` with 17 nodes (each a Python function representing a pipeline step) connected by directed edges. The key innovation is `AnalysisWorkflowState` (a `TypedDict`) — a shared blackboard that every node reads and writes. A conditional edge enables a feedback loop: if the CriticReviewAgent rejects insights, the graph loops back to InsightGenerationAgent for up to 2 revision cycles. The compiled graph runs asynchronously via `astream()`, yielding each node's output as it completes — enabling real-time SSE streaming.

---

**Q3: What is DuckDB and why was it chosen?**

> **A:** DuckDB is an in-process analytical SQL database optimized for OLAP workloads. Chosen because: (1) runs entirely in-memory with no server required; (2) integrates natively with Pandas DataFrames; (3) extremely fast for columnar aggregation queries; (4) supports full SQL including CTEs and window functions. SQLite would be row-oriented and much slower for analytical aggregations.

---

**Q4: How does the application handle multiple users simultaneously?**

> **A:** Each request generates a unique `dataset_id` (UUID) and the pipeline runs independently per request. The DuckDB manager creates separate table names for each dataset (`dataset_{uuid}`). FastAPI's async design helps manage I/O-level concurrency. However, there is a shared in-memory DuckDB connection, which is a limitation under very high concurrency — separate connections per worker would be needed at true scale.

---

**Q5: How does SSE streaming work in this project?**

> **A:** SSE is a one-way push protocol where the server streams events over HTTP. The `/api/analyze/stream` endpoint returns a `StreamingResponse` with `media_type="text/event-stream"`. The generator `event_generator_from_graph()` uses LangGraph's `astream()` to iterate through pipeline nodes and calls `format_sse_event()` to format each as `event: {type}\ndata: {json}\n\n`. On the frontend, since `EventSource` doesn't support POST, the app uses `fetch()` with `response.body.getReader()` for manual stream parsing.

---

**Q6: Why are all data contracts defined as Pydantic models?**

> **A:** Pydantic v2 provides: (1) automatic validation and type coercion; (2) structured output from LLM providers — both Groq and Gemini are asked to return JSON matching the Pydantic schema; (3) safe parsing with `model_validate_json()`; (4) type-safe serialization with `model_dump_json()`. There are 13 model files covering every pipeline artifact from profile through report.

---

### 🟡 Intermediate Level

**Q7: How does the multi-provider LLM failover work?**

> **A:** `LLMRouter` maintains priority-ordered providers: 3 Groq keys → 2 Gemini keys → MockProvider. For each call, `_get_available_providers()` filters out keys on cooldown (`time.time() < cooldowns[key_id]`). The `complete()` method tries each provider. If `LLMProviderError` is raised with "rate", "429", "quota", or "limit" in the message, `mark_cooldown()` stores a 60-second expiration. If all external providers fail, the system falls back to `MockLLMProvider` — which always succeeds and returns synthetic but structurally valid data. This ensures the application never crashes due to LLM unavailability.

---

**Q8: How does the SQL safety validation work? What attacks does it prevent?**

> **A:** `SQLValidator` implements 6 layers: (1) **Multi-statement injection** — rejects queries with semicolons inside; (2) **Statement type restriction** — only `SELECT`/`WITH` allowed; (3) **Keyword blocklist** — 20+ DDL/DML keywords like `DROP`, `DELETE`, `ATTACH`, `PRAGMA`, `EXEC`; (4) **Function blocklist** — filesystem functions like `read_csv`, `httpfs`, `glob`; (5) **URL/protocol detection** — blocks `https://`, `s3://`, `gcs://`, `file://`; (6) **Table name verification** — auto-corrects generic `FROM dataset` to actual table name. This prevents SQL injection, filesystem access, DuckDB extension loading, and cross-table data exfiltration.

---

**Q9: Explain the Critic Review Loop mechanism.**

> **A:** After InsightGenerationAgent generates 3–6 insights, CriticReviewAgent performs an adversarial audit at `temperature=0.0` (deterministic). It compares each insight's claims against ground-truth computed evidence using a 10-point checklist: claim-evidence alignment, causation vs correlation, statistical vs practical significance, proper skewness direction, Pareto threshold (≥75–80%). If `approved=False`, `revise_insights_node` constructs a structured critique listing each rejected claim's reason and required correction, then routes back to `generate_insights_node` which includes the critique in its user prompt. This runs a maximum of 2 times. If still unapproved, flagged insights get confidence downgraded to `"Caveat"` with a disclaimer appended.

---

**Q10: How does the anti-hallucination system work in the planning agent?**

> **A:** `AnalysisPlanningAgent.validate_and_clean_plan()` runs after the LLM returns an `AnalysisPlan`. It does case-insensitive lookup of every column reference against the real profile's column names. Hallucinated numeric columns are removed; if the result is empty, it falls back to the first 4 real numeric columns. Both columns in correlation pairs must be verified numerics. Group-by group column must be in valid columns, metric in numerics. SQL goal `columns_needed` with non-existent columns are dropped. This guarantees zero hallucinated column names reach the statistical or SQL pipeline.

---

**Q11: How does the data cleaning pipeline handle different types of dirty data?**

> **A:** The `DataCleaner` applies 12 sequential passes. Key highlights: (1) **Null placeholder detection** — 30+ string variants mapped to `np.nan` before any downstream processing; (2) **Encoding repair** — `MOJIBAKE_MAP` dict fixes common UTF-8 garbling; (3) **Categorical normalization** — domain-specific maps for directions, devices, channels, genders, tiers, plus `difflib` fuzzy matching to unify typos like `"Electroniks"` → `"Electronics"`; (4) **Range validation** — IQR-based outlier capping; (5) **Imputation** — median/mode with a full audit log. The entire pipeline is unconditional — it runs on every dataset.

---

**Q12: How does the frontend parse Server-Sent Events manually?**

> **A:** Since `EventSource` API only supports GET, the frontend uses `fetch()` POST, gets a streaming `Response`, and calls `response.body.getReader()`. In the loop: `reader.read()` gives chunks (Uint8Array), decoded with `TextDecoder`. Text is appended to a `buffer` string. SSE messages are delimited by `\n\n`, so `buffer.split('\n\n')` gives complete events. The last element (potentially incomplete) stays in the buffer for the next iteration. Each event block is split on `\n` to extract the `event:` and `data:` fields, then `JSON.parse()` deserializes the payload.

---

**Q13: How does the two-tier caching system work?**

> **A:** `ReportBuilder` implements a class-level in-memory dict `_cached_reports` and disk cache at `.cache/reports/{dataset_id}.json`. On write: report saved to both memory and disk (Pydantic `model_dump_json(indent=2)`). On read: memory checked first; if missing, disk deserialized with `model_validate_json()`; if still missing, returns `None`. Similarly for cleaned DataFrames: `.cache/cleaned/{dataset_id}.csv`. On cold start (server restart), disk cache enables seamless report retrieval. `list_history()` scans disk. `clear_all_caches()` uses `shutil.rmtree` on both directories.

---

### 🔴 Advanced Level

**Q14: How would you scale this application to handle 100 concurrent users?**

> **A:** Key changes: (1) **DuckDB isolation** — per-worker DuckDB instances (one per Uvicorn process); (2) **Shared state** — move in-memory caches to Redis or S3 for multi-worker support; (3) **Task queue** — Celery/Dramatiq for background pipeline execution; SSE replaced by WebSocket or polling endpoint; (4) **More LLM keys** — 100 concurrent LLM calls would exhaust Groq free tier; need more keys or paid tier; (5) **Horizontal scaling** — multiple FastAPI instances behind a load balancer; (6) **Memory management** — 100 concurrent DataFrames could be gigabytes; streaming load on-demand.

---

**Q15: What are the security implications of running LLM-generated SQL on a production database?**

> **A:** Risks and mitigations: (1) **Cross-table access** — all uploaded data shares the same DuckDB connection; mitigation: per-request isolated DuckDB instances; (2) **SQL injection via table names** — mitigated by `re.sub(r"[^a-zA-Z0-9_]", "_", dataset_id)` for table names; (3) **DuckDB extension loading** — `INSTALL`/`LOAD` keywords blocked; (4) **Filesystem access** — `read_csv`, `httpfs`, `glob` blocked; (5) **Resource exhaustion** — no query timeouts; a full scan on huge datasets could stall; mitigation: DuckDB `SET threads=...` and query timeouts; (6) The self-healing fallback generates server-side SQL, bypassing LLM — these are deterministic and safe.

---

**Q16: Explain the TypedDict design choice for `AnalysisWorkflowState` vs a Pydantic model.**

> **A:** `TypedDict` was chosen over Pydantic for two reasons: (1) **LangGraph compatibility** — LangGraph's `StateGraph` merges node outputs into the state dict by shallow merging, which aligns naturally with TypedDict; (2) **Performance** — Pydantic validation runs on every field assignment; with 17 nodes and large objects (DataFrames, hundreds of insight objects), avoiding repeated validation overhead matters. `total=False` lets nodes add fields incrementally. The trade-off is no runtime type validation for state fields — mitigated by strong Pydantic types on each service's return values.

---

**Q17: How does the `InsightItem` model handle backward compatibility between LLM response schemas?**

> **A:** `InsightItem` uses `@model_validator(mode="before")` → `sync_aliases()`. Before Pydantic validates fields, this checks paired field groups and syncs them: if `evidence` is missing but `supporting_evidence` exists, it copies one to the other; same for `implication`/`recommendation` and `finding`/`empirical_answer`. If `question_answered` is absent, it's auto-generated from `category` and `title`. If `interpretation` is missing, it's populated from `finding`. This handles different field names from different LLM model versions or providers without requiring prescriptive prompts.

---

**Q18: Walk through exactly how a Groq rate-limit error triggers the fallback mechanism.**

> **A:** Groq API returns 429 → Groq SDK raises exception → `GroqProvider.generate_structured()` catches `Exception as e`, wraps as `LLMProviderError(f"Groq API error: {str(e)}")`, re-raises → `LLMRouter.complete()` catches `LLMProviderError as pe` → checks `"429" in err_str or "rate" in err_str` → True → `mark_cooldown("groq_key_1")` stores `cooldowns["groq_key_1"] = time.time() + 60.0` → loop continues to `groq_key_2` → if all Groq and Gemini keys are exhausted → `except` after loop → `self.mock_provider.generate_structured(...)` → always returns valid synthetic Pydantic model → pipeline continues uninterrupted.

---

**Q19: How does the `PDFExporter` achieve accurate "Page X of N" pagination?**

> **A:** ReportLab's single-pass rendering doesn't know total page count during page 1. `PDFExporter` uses custom `NumberedCanvas` overriding `canvas.Canvas`. The `showPage()` override captures canvas state dict to `_saved_page_states` instead of flushing. `save()` knows `num_pages = len(self._saved_page_states)`, iterates all saved states, restores each (replaying the page), calls `draw_page_number(num_pages)` writing `"Page X of N"` in footer, then flushes each page. This is a classic two-pass rendering pattern.

---

**Q20: What is the purpose of `clean_nan_and_inf()` in the SSE events module?**

> **A:** Python's `float('nan')` and `float('inf')` are valid Python values but **invalid JSON** — `json.dumps(float('nan'))` produces `NaN` which `JSON.parse()` throws `SyntaxError` for. Statistical computations (scipy, pandas) frequently produce NaN (empty correlations, missing standard deviations) and Inf (division by zero). `clean_nan_and_inf()` recursively walks data structures and replaces any `math.isnan()` or `math.isinf()` floats with `None`, which serializes as JSON `null` and React handles gracefully.

---

**Q21: How does `SemanticRoleClassifier` avoid misclassifying numeric IDs as measures?**

> **A:** Multiple guards: (1) **Name regex** — `IDENTIFIER_REGEX` explicitly matches `id`, `_id`, `.*_id`, `.*_key`, `.*_uuid`, `order_id`, `customer_id`, etc. — matched columns → `identifier` regardless of dtype; (2) **Unique ratio** — if ≥95% of values are unique AND dtype is NOT numeric/float, it's an identifier; (3) **Numeric dtype protection** — unique ratio check explicitly excludes `semantic_type="numeric"` and float/double dtypes, so `age` with 50 unique values won't be mislabeled; (4) **Rate measure detection** — columns with names matching `rate|score|ratio|margin|pct|percent|gpa|avg` → `measure_rate` (AVG, not SUM).

---

**Q22: How would you add a new agent to the pipeline?**

> **A:** Four steps: (1) Create the agent class in `backend/app/agents/` with a `@classmethod` accepting state fields and returning a Pydantic model; (2) Create the Pydantic model in `backend/app/models/`; (3) Add the field to `AnalysisWorkflowState` in `orchestration/state.py`; (4) In `graph.py`: add `graph.add_node("my_new_agent", my_new_agent_node)` and appropriate `graph.add_edge()` calls. Also add entry to `STEP_METADATA` in `events.py` and update `TOTAL_PIPELINE_STEPS`. SSE streaming and progress tracking automatically pick up the new step.

---

**Q23: Why does the Critic agent run at `temperature=0.0` while others use `0.1`?**

> **A:** Temperature controls randomness. `temperature=0.0` makes the model deterministic — same input always produces the same output. The `CriticReviewAgent`'s job is fact-checking against ground-truth computed evidence. We want its verdict **consistent and reproducible** — not influenced by random sampling. If the critic used temperature 0.1, it might approve insights in one run and reject them in another for identical data. Using 0.0 means the critic behaves like a deterministic rule evaluator, ensuring reliable quality gates. Insight generation uses 0.1 because creative variation in finding different angles is acceptable.

---

## 9. Project Pitches

### 30-Second Pitch

> "DataPilot is an AI-powered data analysis platform where you upload a CSV or Excel file and get a complete, professional analysis in real time — automatic data cleaning, statistical insights, interactive charts, AI-generated business findings, and a downloadable PDF report. It's powered by a 17-step multi-agent pipeline built on LangGraph, where specialized AI agents coordinate to understand your data, plan the analysis, write and validate SQL queries, generate evidence-grounded insights, and self-correct using an adversarial critic review loop. The entire pipeline streams live progress to the frontend using Server-Sent Events."

---

### 2-Minute Pitch

> "DataPilot is a full-stack, production-ready data analysis assistant built to demonstrate what's possible with modern AI orchestration. The core is a 17-node LangGraph pipeline where each node is a specialized agent or service.
>
> When you upload a file, the system validates and ingests it into an in-memory DuckDB database — chosen because it's a columnar analytical engine that integrates natively with Pandas and is blazing fast for aggregation queries. Then it runs an enterprise-grade 12-section data cleaning pipeline — handling encoding issues, null imputation, outlier correction, categorical normalization via domain-specific abbreviation maps, and typo collapsing using difflib fuzzy matching.
>
> After cleaning, an LLM agent understands the domain and business context, a planning agent creates a validated analysis blueprint (post-validating all column references to eliminate hallucinations), a statistical engine computes univariate metrics and correlations, and an SQL agent generates and executes safe DuckDB queries validated by a 6-layer security checker.
>
> The most interesting part is the insight generation loop: an InsightGenerationAgent creates 3–6 evidence-backed findings, then an adversarial CriticReviewAgent at temperature 0 audits them against a 10-point fact-checking checklist — catching causation-correlation confusion, skewness misinterpretation, false Pareto claims. If the critic rejects claims, insights are revised with structured feedback and re-audited, up to 2 times.
>
> The frontend is a React app consuming real-time SSE events using fetch() + ReadableStreamDefaultReader, showing a live 17-step pipeline tracker, then rendering a full dashboard with quality audit, statistics, Plotly visualizations, AI insights, and the full report. The system supports multi-provider LLM failover across Groq and Gemini with automatic rate-limit cooldowns and PII privacy filtering."

---

### 5-Minute Pitch

> "I'd like to walk you through DataPilot, a multi-agent AI data analysis assistant I built to solve a real problem: business analysts spend most of their time on repetitive data preparation and reporting rather than actual decision-making.
>
> **Architecture:** The system is built around a 17-node LangGraph StateGraph — a directed graph where each node is either a deterministic service or an LLM-powered agent. They share a single `AnalysisWorkflowState` TypedDict that acts as a blackboard, letting each node read prior results and write its outputs without tight coupling.
>
> **Ingestion and Cleaning:** When you upload a CSV or Excel file, Node 1 validates the file (size, extension, filename sanitization). Node 2 ingests it into an in-memory DuckDB table using `CREATE OR REPLACE TABLE AS SELECT * FROM temp_df_view`. Nodes 3 and 4 profile and clean the data through a 12-section enterprise pipeline: null placeholder detection across 30+ string variants, encoding repair via mojibake maps, categorical normalization using domain-specific abbreviation dictionaries, IQR-based outlier capping, and median/mode imputation with a full audit trail — all unconditionally applied.
>
> **The AI Agents:** Node 5, the DatasetUnderstandingAgent, sends a compact column summary to an LLM and infers business domain, primary entity, candidate KPIs, and 3–5 business questions. Node 6, the AnalysisPlanningAgent, generates an analysis blueprint and critically post-validates every column reference against the real profile to eliminate hallucinated column names. Node 8, the SQLGenerationAgent, translates the plan into DuckDB SQL with 9 strict rules in the system prompt — no DDL, no mutations, only verified column names. The SQL is then validated by a 6-layer security checker and executed by the SQLExecutor with self-healing fallback synthesis if the primary query fails.
>
> **The Critic Loop:** The most interesting part is Nodes 13–15. The InsightGenerationAgent produces 3–6 insights following an 8-step reasoning chain: Data → Analysis → Finding → Evidence → Interpretation → Business Impact → Recommendation → Confidence. Then the CriticReviewAgent, at temperature 0 for determinism, audits against a 10-point checklist: claim-evidence alignment, correlation-causation confusion, statistical vs practical significance, proper skewness, Pareto threshold verification. If any claim fails, insights are revised with structured critique injected into the next prompt — up to 2 cycles. Persistent failures get a 'Caveat' confidence with a disclaimer.
>
> **Reliability Engineering:** The LLM router implements multi-provider failover across 3 Groq keys and 2 Gemini keys, with 60-second rate-limit cooldowns and graceful fallback to a MockLLMProvider that always succeeds. Before any messages reach external LLMs, a DataPrivacyFilter regex-redacts emails, phone numbers, SSNs, credit cards, and API keys.
>
> **Frontend:** The React frontend uses fetch() with ReadableStreamDefaultReader — not EventSource, because SSE doesn't support POST — for manual chunked SSE parsing. It shows a live 17-step pipeline tracker with preview cards, then a full dashboard with quality, statistics, Plotly visualizations, AI insights, the full markdown report, and cleaned data with before/after diff view and CSV/Excel download.
>
> **Production Readiness:** Deployed on Render.com with a YAML config. Two-tier memory + disk caching for session persistence. 22 pytest test files. The system degrades gracefully at every failure point — from SQL self-healing to LLM provider failover to critic caveat downgrade."

---

## 10. Improvements & Limitations

### Current Limitations

| Area | Limitation |
|---|---|
| **Concurrency** | Single shared DuckDB connection — not thread-safe under high concurrency; cross-request table access theoretically possible |
| **Stateful Backend** | In-memory caches are per-process — breaks with multiple Uvicorn workers or load-balanced instances |
| **Session Persistence** | Reports lost on server restart if disk cache unavailable; no user authentication |
| **File Size** | Hard cap at 10 MB — large enterprise datasets unsupported |
| **Blocking LLM Calls** | LLM calls are synchronous within async nodes — a slow LLM call blocks the SSE stream for that node |
| **Single-file Analysis** | No support for joining multiple files or multi-table queries |
| **No User Auth** | Any user can access any dataset by knowing the UUID |
| **Date Handling** | Complex multi-format date normalization may fail on uncommon locale-specific formats |
| **Mock Provider Quality** | Returns simplified synthetic data — degraded analysis quality in no-API-key mode |
| **No Chart Customization** | Users cannot configure chart types, colors, or axes |

### Potential Bugs

| Bug | Description |
|---|---|
| **DuckDB Race Condition** | Two simultaneous requests with the same `dataset_id` could overwrite each other's table |
| **SSE Payload Size** | Very large report objects in the `complete` event could exceed browser memory when JSON-serialized |
| **Mock Critic Always Approves** | If MockProvider is used for both insights and critic, it always returns `approved=True`, bypassing the revision mechanism |
| **History Stale Data** | After deletion, frontend localStorage and backend may desync |
| **Excel Datetime Precision** | `openpyxl` loads dates as Python `datetime` objects; column sanitization converts them to strings, losing timezone info |

### Missing Tests

| Component | Gap |
|---|---|
| `ChartGenerator` | No tests for domain-specific chart type selection logic |
| `ReportBuilder.build_report_from_dataset()` | No integration test for full on-demand rebuild pipeline |
| `LLMRouter` cooldown | No test verifying cooled-down key is skipped in subsequent calls |
| Frontend components | No Jest/Vitest tests for any React component |
| SSE parsing | No end-to-end test verifying frontend SSE parser handles partial chunk boundaries |

### Possible Improvements

| Improvement | Impact |
|---|---|
| **Per-request DuckDB connections** | Fix concurrency; isolated in-memory DB per request |
| **Redis for caching** | Enable horizontal scaling with shared state across workers |
| **Celery task queue** | Background pipeline execution; polling instead of SSE for better reliability |
| **User authentication** | JWT-based auth + per-user dataset isolation |
| **Streaming LLM calls** | Use streaming completions to reduce perceived latency for long LLM responses |
| **Multi-file join** | Upload multiple files, analyze relationships between them |
| **Natural language querying** | Chat interface for follow-up questions triggering targeted SQL generation |
| **Chart export** | Individual chart PNG/SVG export from Visualizations tab |
| **Column-level cleaning control** | Configure cleaning rules per column (e.g., imputation strategy) |
| **Webhook/email delivery** | Send completed reports via email or webhook |
| **CI/CD pipeline** | GitHub Actions for automated testing and Render deployment |
| **Observability** | Integrate OpenTelemetry or Langfuse for LLM call tracing |

---

*Generated from full source code audit — October 2026*  
*Project: `multi-agent-ai-data-analysis-assistant`*
