# DataPilot: Multi-Agent Data Analyst — Complete Workflow Document

> **Purpose:** This document traces the entire execution of the DataPilot project based strictly on the actual source code. Every file path, function name, class, and behavior described here has been confirmed from the codebase. If something is not implemented in the code, it is not mentioned as an implemented feature.

---

## Table of Contents

1. [Project Structure Overview](#1-project-structure-overview)
2. [Application Entry Point and Startup](#2-application-entry-point-and-startup)
3. [Frontend Entry Point](#3-frontend-entry-point)
4. [Dataset Upload Flow (Two Paths)](#4-dataset-upload-flow-two-paths)
5. [The 17-Node LangGraph Pipeline](#5-the-17-node-langgraph-pipeline)
6. [Agent Implementations — Detailed Trace](#6-agent-implementations--detailed-trace)
7. [LLM Router and Provider Failover](#7-llm-router-and-provider-failover)
8. [State Flow Throughout the Pipeline](#8-state-flow-throughout-the-pipeline)
9. [SSE Streaming to the Frontend](#9-sse-streaming-to-the-frontend)
10. [Visualization Flow](#10-visualization-flow)
11. [PDF Export Flow](#11-pdf-export-flow)
12. [Error Handling Flow](#12-error-handling-flow)
13. [File-by-File Connection Map](#13-file-by-file-connection-map)
14. [Master Execution Flow Diagram](#14-master-execution-flow-diagram)
15. [How to Explain This Project in an Interview](#15-how-to-explain-this-project-in-an-interview)

---

## 1. Project Structure Overview

```
multi-agent-ai-data-analysis-assistant/
├── backend/
│   ├── __init__.py                     ← sys.path bootstrapper
│   └── app/
│       ├── main.py                     ← FastAPI application entry point
│       ├── core/
│       │   ├── config.py               ← Pydantic Settings (env vars)
│       │   ├── exceptions.py           ← Custom exception hierarchy
│       │   └── logging.py              ← Logger configuration
│       ├── api/
│       │   └── routes/
│       │       ├── upload.py           ← POST /api/upload
│       │       ├── analysis.py         ← POST /api/analyze/stream (SSE)
│       │       ├── profile.py          ← GET /api/dataset/{id}/profile & quality
│       │       ├── report.py           ← GET/POST /api/dataset/{id}/report & PDF
│       │       └── health.py           ← GET /api/health
│       ├── orchestration/
│       │   ├── state.py                ← AnalysisWorkflowState TypedDict
│       │   ├── events.py               ← STEP_METADATA, format_sse_event()
│       │   └── graph.py                ← 17-node LangGraph StateGraph
│       ├── agents/
│       │   ├── understand_dataset.py   ← DatasetUnderstandingAgent (LLM)
│       │   ├── plan_analysis.py        ← AnalysisPlanningAgent (LLM)
│       │   ├── generate_sql.py         ← SQLGenerationAgent (LLM)
│       │   ├── generate_insights.py    ← InsightGenerationAgent (LLM)
│       │   ├── critic_review.py        ← CriticReviewAgent (LLM)
│       │   ├── revise_insights.py      ← InsightRevisionOrchestrator
│       │   └── generate_report.py      ← ReportGenerationAgent (LLM)
│       ├── llm/
│       │   ├── base.py                 ← LLMProvider ABC
│       │   ├── router.py               ← LLMRouter (failover logic)
│       │   ├── groq_provider.py        ← GroqProvider (llama-3.3-70b)
│       │   ├── gemini_provider.py      ← GeminiProvider (gemini-2.5-flash)
│       │   ├── mock_provider.py        ← MockLLMProvider (fallback)
│       │   └── privacy.py              ← DataPrivacyFilter (PII redaction)
│       ├── services/
│       │   ├── ingestion/
│       │   │   ├── validator.py        ← FileValidator
│       │   │   ├── loader.py           ← DatasetLoader
│       │   │   └── duckdb_manager.py   ← DuckDBManager (in-memory DB)
│       │   ├── profiling/
│       │   │   └── profiler.py         ← DatasetProfiler (deterministic)
│       │   ├── quality/
│       │   │   └── checker.py          ← QualityChecker (deterministic)
│       │   ├── cleaning/
│       │   │   └── cleaner.py          ← DataCleaner (deterministic)
│       │   ├── statistics/
│       │   │   └── engine.py           ← StatisticalEngine (deterministic)
│       │   ├── sql/
│       │   │   ├── validator.py        ← SQLValidator (AST security)
│       │   │   └── executor.py         ← SQLExecutor (DuckDB runner)
│       │   ├── patterns/
│       │   │   └── detector.py         ← PatternDetector (deterministic)
│       │   ├── visualization/
│       │   │   ├── chart_generator.py  ← ChartGenerator (Plotly specs)
│       │   │   └── role_classifier.py  ← SemanticRoleClassifier
│       │   └── reporting/
│       │       ├── report_builder.py   ← ReportBuilder (cache + pipeline)
│       │       └── pdf_exporter.py     ← PDFExporter (ReportLab)
│       └── models/                     ← Pydantic data models for all artifacts
└── frontend/
    ├── index.html
    └── src/
        ├── main.jsx                    ← React app entry (ReactDOM.createRoot)
        ├── App.jsx                     ← Entire frontend state machine (952 lines)
        ├── services/
        │   └── apiConfig.js            ← Dynamic backend URL resolver
        └── components/
            ├── PipelineTracker.jsx     ← Live 17-step progress tracker
            ├── OverviewTab.jsx         ← Dashboard overview panel
            ├── DataQualityTab.jsx      ← Quality score panel
            ├── StatisticsTab.jsx       ← Statistical analysis display
            ├── VisualizationsTab.jsx   ← Plotly chart renderer
            ├── InsightsTab.jsx         ← Evidence-backed insights display
            ├── ReportMarkdownTab.jsx   ← Full 13-section markdown report
            ├── CleanDataTab.jsx        ← Cleaned dataset preview + downloads
            ├── HistorySidebar.jsx      ← Analysis history sidebar
            └── ChartRenderer.jsx       ← Individual Plotly chart wrapper
```

---

## 2. Application Entry Point and Startup

**File:** `backend/app/main.py`

### Startup Sequence

```
Python process starts
        |
backend/__init__.py         <- sys.path setup so "backend.*" imports resolve
        |
backend/app/main.py         <- FastAPI application creation
        |
Settings loaded             <- backend/app/core/config.py (reads .env)
        |
CORS middleware attached     <- from settings.CORS_ORIGINS
        |
Exception handlers registered (AppBaseException -> 400, Exception -> 500)
        |
Routers included:
  /api/health       <- health_router
  /api/upload       <- upload_router
  /api/dataset/*    <- profile_router, report_router
  /api/analyze/*    <- analysis_router
        |
duckdb_manager = DuckDBManager.get_instance()   <- singleton, in-memory DuckDB
llm_router     = LLMRouter.get_instance()       <- singleton, provider pool
        |
Server ready on HOST:PORT (default 0.0.0.0:8000)
```

### Step-by-Step Explanation

**Step 1 — `backend/__init__.py`**

This file adds the project root to `sys.path` so that imports like `from backend.app.core.config import settings` work from anywhere in the project.

**Step 2 — `backend/app/main.py`**

The `lifespan` async context manager (lines 27-31) is registered with FastAPI. On startup it logs the app name and version. No data is loaded here — the app starts clean.

**Step 3 — `backend/app/core/config.py`**

Class `Settings` (extends `pydantic_settings.BaseSettings`) reads environment variables from `.env`. Key settings:
- `GROQ_API_KEY_1/2/3` — up to 3 Groq API keys for load balancing
- `GEMINI_API_KEY_1/2` — up to 2 Gemini keys
- `GROQ_MODEL = "llama-3.3-70b-versatile"`
- `GEMINI_MODEL = "gemini-2.5-flash"`
- `MAX_UPLOAD_SIZE_MB = 10`, `MAX_DATASET_ROWS = 500000`, `MAX_DATASET_COLUMNS = 500`

**Step 4 — `backend/app/services/ingestion/duckdb_manager.py`**

On module import, `duckdb_manager = DuckDBManager.get_instance()` creates a singleton with an **in-memory DuckDB connection** (`duckdb.connect(database=":memory:")`). All uploaded datasets are stored here as tables during the session.

**Step 5 — `backend/app/llm/router.py`**

On module import, `llm_router = LLMRouter.get_instance()` creates a singleton managing the LLM provider pool with cooldown tracking.

---

## 3. Frontend Entry Point

**File:** `frontend/src/main.jsx`

```jsx
ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```

This mounts the single-page React application. The entire frontend is contained in one state machine in `App.jsx`.

### Frontend State Machine (`App.jsx`)

The `stage` state variable drives what the user sees:
- `'upload'` — file selection and upload screen
- `'streaming'` — pipeline running with live step tracker
- `'dashboard'` — results tabs with charts, insights, report

**API URL Resolution** (`frontend/src/services/apiConfig.js`, function `getApiBaseUrl()`):

The backend URL is resolved in this priority order:
1. User-saved override in `localStorage` (key: `datapilot_api_base_url`)
2. Build-time `VITE_API_BASE_URL` environment variable
3. Intelligent cloud inference: if hosted on `*.onrender.com`, auto-target `-backend.onrender.com`
4. Default fallback: `http://localhost:8000`

On startup, `App.jsx` calls `GET /api/dataset/history` to sync any previously cached reports into local `history` state, which is also persisted to `localStorage` under `data_analysis_history`.

---

## 4. Dataset Upload Flow (Two Paths)

The project supports **two distinct upload paths**:

| Path | Endpoint | Purpose |
|------|----------|---------|
| Path A | `POST /api/upload` | Simple upload + schema inspection only. Returns metadata + preview. Does NOT trigger the AI pipeline. |
| Path B | `POST /api/analyze/stream` | Uploads file AND immediately runs the full 17-node multi-agent pipeline with SSE streaming. This is the **main analysis path**. |

---

### Path A — Direct Upload via `/api/upload`

**File:** `backend/app/api/routes/upload.py`
**Function:** `upload_dataset(file: UploadFile)` (line 19)

```
User sends POST /api/upload with multipart file
        |
Step 1: content = await file.read()
        |
Step 2: FileValidator.validate_file_metadata(filename, file_size, content_type)
        |       backend/app/services/ingestion/validator.py
        |       - sanitize_filename(): strips path, replaces special chars with _
        |       - checks extension in {.csv, .xlsx, .xls}
        |       - checks file_size > 0 and <= MAX_UPLOAD_SIZE_MB * 1024 * 1024
        |       Returns: (sanitized_name, extension)
        |       On failure: raises FileValidationError -> HTTP 400
        |
Step 3: DatasetLoader.load_and_sanitize(content, extension)
        |       backend/app/services/ingestion/loader.py
        |       - For .csv: tries 5 encodings, sniffs delimiters
        |       - For .xlsx/.xls: uses openpyxl or xlrd
        |       - Validates row/column counts vs limits
        |       - Sanitizes column names (SQL-safe identifiers)
        |       Returns: (pd.DataFrame, List[ColumnSchema])
        |       On failure: raises IngestionError -> HTTP 422
        |
Step 4: duckdb_manager.register_dataframe(df)
        |       backend/app/services/ingestion/duckdb_manager.py
        |       - Generates dataset_id = uuid.uuid4()
        |       - Creates DuckDB table: CREATE OR REPLACE TABLE dataset_<uuid> AS SELECT * FROM temp_df_view
        |       - Stores mapping: registered_tables[dataset_id] = table_name
        |       Returns: (dataset_id, table_name)
        |
Step 5: Save raw bytes to backend/temp_uploads/{dataset_id}_{filename}
        |
Step 6: duckdb_manager.get_preview_rows(table_name, limit=5)
        |       Executes: SELECT * FROM {table_name} LIMIT 5
        |       Returns: List[Dict]
        |
Returns UploadResponse:
  { success: True, dataset: DatasetMetadata { dataset_id, table_name, filename,
    file_size_bytes, row_count, column_count, columns, preview_rows, upload_timestamp } }
```

---

### Path B — SSE Analysis via `/api/analyze/stream`

**This is the primary path.** When the user clicks "Analyze" in the frontend, `handleStartAnalysis()` in `App.jsx` (line 198) is called.

```
User clicks "Analyze" button in frontend (App.jsx, handleStartAnalysis())
        |
setStage('streaming'), setCurrentStep('validate_file')
abortControllerRef.current = new AbortController()
        |
formData.append('file', selectedFile)
        |
fetch(`${apiBase}/api/analyze/stream`, { method: 'POST', body: formData, signal: abortController.signal })
        |
backend/app/api/routes/analysis.py
analyze_file_stream(file: UploadFile)
  - content = await file.read()
  - dataset_id = str(uuid.uuid4())
  - initial_state = { dataset_id, filename, file_bytes: content, logs: [], revision_count: 0, critic_approved: False }
  - Returns: StreamingResponse(event_generator_from_graph(initial_state), media_type="text/event-stream")
        |
event_generator_from_graph(initial_state)   <- async generator
  - Emits SSE: pipeline_start event
  - Calls: analysis_workflow_graph.astream(current_state)   <- LangGraph async stream
  - For each node output: emits SSE step_complete event
  - After all 17 nodes: emits SSE complete event with full report JSON
        |
Frontend App.jsx reads SSE stream:
  while (true) { const {done, value} = await reader.read(); ... }
  Parses event types: step_complete, pipeline_complete, pipeline_error
```

---

## 5. The 17-Node LangGraph Pipeline

**File:** `backend/app/orchestration/graph.py`
**Function:** `build_analysis_graph()` (line 376)
**Compiled object:** `analysis_workflow_graph` (line 432)

The entire pipeline is a **LangGraph `StateGraph`** compiled from 17 nodes. All nodes share the `AnalysisWorkflowState` TypedDict (defined in `backend/app/orchestration/state.py`). Each node receives the complete state and returns a dict of fields to update.

### Node Sequence and Routing

```
START
  |
  [1] validate_file            -> [2] load_dataset
  [2] load_dataset             -> [3] profile_and_audit
  [3] profile_and_audit        -> [4] clean_and_standardize
  [4] clean_and_standardize    -> [5] understand_dataset
  [5] understand_dataset       -> [6] plan_analysis
  [6] plan_analysis            -> [7] run_statistical_analysis
  [7] run_statistical_analysis -> [8] generate_sql
  [8] generate_sql             -> [9] validate_sql
  [9] validate_sql             -> [10] execute_sql
  [10] execute_sql             -> [11] detect_patterns
  [11] detect_patterns         -> [12] render_charts
  [12] render_charts           -> [13] generate_insights
  [13] generate_insights       -> [14] critic_review
  [14] critic_review           -> [15] revise_insights
  [15] revise_insights --------> CONDITIONAL (should_continue_revision):
         |
         |- critic_approved=False AND revision_count <= 2  -> [13] generate_insights  (LOOP BACK)
         |
         `- critic_approved=True OR revision_count > 2     -> [16] generate_report
              |
             [16] generate_report -> [17] render_pdf -> END
```

### Conditional Edge — `should_continue_revision()`

**File:** `backend/app/orchestration/graph.py`, function `should_continue_revision(state)` (line 325)

```python
def should_continue_revision(state) -> Literal["generate_insights", "generate_report"]:
    if not state.get("critic_approved", True) and state.get("revision_count", 0) <= 2:
        return "generate_insights"
    return "generate_report"
```

This creates a **feedback loop**: if the `CriticReviewAgent` rejects the insights AND fewer than 2 revision loops have occurred, execution routes back to `generate_insights_node` with the critic's feedback attached. Maximum 2 revision loops are enforced.

---

### Node 1 — `validate_file_node`

**File:** `backend/app/orchestration/graph.py` (line 30)

**Input from state:** `state["filename"]`, `state["file_bytes"]`

**Processing:** Calls `FileValidator.validate_file_metadata(filename, len(file_bytes))` from `backend/app/services/ingestion/validator.py`

**Output to state:** `{ "filename": sanitized_name, "file_extension": ext, "current_step": "validate_file", "step_index": 1, "total_steps": 17 }`

---

### Node 2 — `load_dataset_node`

**File:** `backend/app/orchestration/graph.py` (line 48)

**Input from state:** `state["file_bytes"]`, `state["file_extension"]`, `state["dataset_id"]`

**Processing:**
1. `DatasetLoader.load_and_sanitize(file_bytes, ext)` from `backend/app/services/ingestion/loader.py` -> `(df, metadata)`
2. Generates raw table name: `raw_dataset_<uuid>`
3. `duckdb_manager.register_dataframe(df, dataset_id, raw_table)` from `backend/app/services/ingestion/duckdb_manager.py`

**Output to state:** `{ "table_name": raw_table, "raw_table_name": raw_table, "raw_df": df, "df": df }`

**Why this matters:** The raw DataFrame is kept separately from the cleaned one so Node 4 can compare before/after.

---

### Node 3 — `profile_and_audit_node`

**File:** `backend/app/orchestration/graph.py` (line 70)

**Input from state:** `state["df"]`, `state["dataset_id"]`, `state["table_name"]`

**Processing:**
1. `DatasetProfiler.profile_dataset(df, dataset_id, table_name)` from `backend/app/services/profiling/profiler.py`
2. `QualityChecker.audit_dataset(df, profile)` from `backend/app/services/quality/checker.py`

Both are **100% deterministic Python/NumPy — no LLM involved.**

**Output to state:** `{ "profile": DatasetProfile, "quality": QualityReport }`

**What `DatasetProfiler.profile_dataset()` computes:**

For each column:
- Counts nulls, unique values
- Calls `infer_semantic_type()`: classifies as `numeric`, `categorical`, `datetime`, `boolean`, or `identifier`
- For numeric: `compute_numeric_stats()` — min, max, mean, median, std, Q25, Q75, IQR, skewness (NumPy percentile + pandas .skew())
- For categorical: `compute_categorical_stats()` — top-5 frequency table, cardinality
- For datetime: `compute_datetime_stats()` — min/max dates, days range
- Also counts duplicate rows across all columns

**What `QualityChecker.audit_dataset()` computes:**

Deterministic checks (no LLM):
- Missing values per column
- Duplicate rows
- Outliers: IQR method (mild: 1.5× IQR, extreme: 3.0× IQR) + Z-score (|z| >= 3.5)
- Inconsistent categories: groups values by lowercase+stripped form, flags casing collisions (e.g., "electronics" vs "Electronics" vs "ELECTRONICS")
- Computes a **quality score 0-100** and **grade A/B/C/D/F**

---

### Node 4 — `clean_and_standardize_node`

**File:** `backend/app/orchestration/graph.py` (line 88)

**Input from state:** `state["raw_df"]`, `state["dataset_id"]`, `state["filename"]`

**Processing:**
1. `DataCleaner.clean_dataset(raw_df, dataset_id, filename)` from `backend/app/services/cleaning/cleaner.py` -> `(cleaned_df, cleaning_summary)`
2. `ReportBuilder.cache_cleaned_df(dataset_id, cleaned_df)` -> stores in memory + writes to `.cache/cleaned/{dataset_id}.csv`
3. Registers cleaned DataFrame to DuckDB as a new table: `dataset_<uuid>` (without `raw_` prefix)
4. Re-profiles: `DatasetProfiler.profile_dataset(cleaned_df, ...)` -> updates `profile` in state

**Output to state:** `{ "df": cleaned_df, "cleaned_df": cleaned_df, "table_name": clean_table, "cleaning_summary": dict, "profile": cleaned_profile }`

**Why this matters:** From this node onward, ALL downstream nodes (statistics, SQL, patterns, charts, insights, report) operate on the CLEANED DataFrame and cleaned DuckDB table. The profile in state is also replaced with the cleaned version.

---

### Node 5 — `understand_dataset_node` — FIRST LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 118)
**Agent:** `backend/app/agents/understand_dataset.py`
**Class:** `DatasetUnderstandingAgent`
**Method:** `analyze(profile, quality, filename)` (line 12)

**Input from state:** `state["profile"]`, `state["quality"]`, `state["filename"]`

**Processing:**

The agent builds a column summary block with each column's name, semantic type, dtype, null%, min/max/mean (for numeric) or top values (for categorical), and 3 sample values.

System prompt: "You are an expert Chief Data Scientist and AI Business Analyst. Your objective is to interpret the real-world business meaning of a dataset based strictly on its schema, column statistics, and data quality metrics. Do NOT hallucinate..."

User prompt includes: filename, total rows/columns, duplicate count, quality score, all column characteristics.

Asks the LLM to determine: domain, dataset narrative, primary entity, KPIs, segmentation dimensions, 3-5 business questions, data limitations.

Calls: `llm_router.complete("understand_dataset", messages, response_model=DatasetUnderstanding, temperature=0.1)`

**LLM returns:** `DatasetUnderstanding` Pydantic model with `domain`, `dataset_summary`, `target_entity`, `key_kpis` (list), `dimension_columns`, `core_questions`, `data_limitations`.

**Output to state:** `{ "understanding": DatasetUnderstanding }`

**This is purely LLM-generated text interpretation. No Python calculation involved.**

---

### Node 6 — `plan_analysis_node` — SECOND LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 133)
**Agent:** `backend/app/agents/plan_analysis.py`
**Class:** `AnalysisPlanningAgent`
**Method:** `plan(profile, understanding)` (line 79)

**Input from state:** `state["profile"]`, `state["understanding"]`

**Processing:**

Provides the LLM with domain, dataset summary, target entity, candidate KPIs, core questions, and the **full verified column lists** (numeric, categorical, datetime, all columns).

Asks for: primary goal, numeric columns for descriptive stats, correlation pairs, 2-4 group-by analysis plans (group_column + metric_column + aggregation + purpose), 2-4 SQL query goals, recommended chart types.

Calls: `llm_router.complete("plan_analysis", messages, response_model=AnalysisPlan, temperature=0.1)`

**Post-LLM validation — `validate_and_clean_plan()`** (line 14):

This critical method runs AFTER the LLM returns. It cross-references every column name mentioned by the LLM against `profile.column_profiles`. Any hallucinated column is removed. Fallback values are inserted if needed. This guarantees zero hallucinated column references reach downstream nodes.

**Output to state:** `{ "plan": AnalysisPlan }`

---

### Node 7 — `run_statistical_analysis_node` — DETERMINISTIC

**File:** `backend/app/orchestration/graph.py` (line 147)
**Service:** `backend/app/services/statistics/engine.py`
**Class:** `StatisticalEngine`
**Method:** `run_analysis(df, profile, plan)` (line 295)

**Input from state:** `state["df"]` (cleaned), `state["profile"]`, `state["plan"]`

**ALL CALCULATIONS ARE PYTHON/NUMPY/SCIPY — NO LLM:**

1. **Univariate Metrics** (`compute_univariate_metrics()`): For each numeric column in `plan.descriptive_numeric_columns`, computes: count, mean, median, std, variance, min, max, p10/p25/p75/p90, IQR, skewness (pandas `.skew()`), kurtosis (pandas `.kurtosis()`). Also generates plain-English description of skewness direction (threshold: >0.5 = right-skewed, <-0.5 = left-skewed).

2. **Correlation Analysis** (`compute_correlations()`): For each [col1, col2] pair in `plan.correlation_pairs`, uses `scipy.stats.pearsonr()` and `scipy.stats.spearmanr()`. Classifies strength (Negligible/Weak/Moderate/Strong/Very Strong based on |r|). Distinguishes statistical significance (p < 0.05) from practical significance.

3. **Group-By Analysis** (`compute_groupby_analyses()`): For each group plan in `AnalysisPlan.group_by_analyses`, groups the cleaned DataFrame. Computes sum, mean, median, min, max, share% per group value. Runs ANOVA F-test (`scipy.stats.f_oneway()`) across groups.

**Output to state:** `{ "statistics": StatisticalAnalysisResult }`

---

### Node 8 — `generate_sql_node` — THIRD LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 162)
**Agent:** `backend/app/agents/generate_sql.py`
**Class:** `SQLGenerationAgent`
**Method:** `generate(profile, plan, table_name)` (line 13)

**Input from state:** `state["profile"]`, `state["plan"]`, `state["table_name"]`

**Processing:**

Sends the LLM: exact DuckDB table name, full column list with semantic types and dtypes, which columns are numeric (safe for SUM/AVG) vs categorical (safe for COUNT), SQL goals from the plan.

Strict rules in system prompt: Must use only SELECT or WITH, must double-quote column names, must not use INSERT/UPDATE/DELETE/DROP/ALTER/CREATE/COPY/ATTACH, no trailing semicolons, reference only verified columns, handle division by zero with NULLIF.

Calls: `llm_router.complete("generate_sql", messages, response_model=SQLGenerationResponse, temperature=0.1)`

**LLM returns:** `SQLGenerationResponse` with list of `GeneratedSQLQuery` objects (name, purpose, sql string).

**Output to state:** `{ "sql_queries": List[GeneratedSQLQuery] }`

---

### Node 9 — `validate_sql_node` — DETERMINISTIC SECURITY GUARD

**File:** `backend/app/orchestration/graph.py` (line 177)
**Service:** `backend/app/services/sql/validator.py`
**Class:** `SQLValidator`
**Method:** `validate_sql(sql, expected_table)` (line 23)

**Input from state:** `state["sql_queries"]`, `state["table_name"]`

**Processing — Pure Python regex/string checks (no LLM):**
1. Strips trailing semicolons; rejects if `;` is inside the query
2. Verifies query starts with `SELECT` or `WITH`
3. Scans for disallowed keywords: INSERT, UPDATE, DELETE, DROP, ALTER, CREATE, COPY, ATTACH, INSTALL, LOAD, PRAGMA, EXEC, TRUNCATE, VACUUM, CHECKPOINT (using word-boundary regex)
4. Scans for disallowed functions: `read_csv`, `read_parquet`, `httpfs`, `glob`, etc.
5. Detects URL schemes: `https://`, `s3://`, `gcs://`, `file://`
6. If expected table missing but `FROM dataset` or `FROM data` found, rewrites to correct table name

Queries that fail are silently dropped with a warning log. Only safe queries pass through.

**Output to state:** `{ "validated_sql_queries": List[GeneratedSQLQuery] }`

---

### Node 10 — `execute_sql_node` — DETERMINISTIC DUCKDB

**File:** `backend/app/orchestration/graph.py` (line 199)
**Service:** `backend/app/services/sql/executor.py`
**Class:** `SQLExecutor`
**Method:** `execute_queries(queries, dataset_id, table_name)`

**Input from state:** `state["validated_sql_queries"]`, `state["dataset_id"]`, `state["table_name"]`

**Processing — Pure DuckDB execution (no LLM):**

For each validated query:
1. `get_table_schema(table_name)` — runs `DESCRIBE table_name` to get actual column types from DuckDB
2. `repair_query_syntax()` — fixes common LLM mistakes (table name used as column, aggregation on string columns, unquoted table names)
3. `duckdb_manager.execute_query(sql)` — runs the query, gets DataFrame back
4. On failure: `synthesize_fallback_query()` generates a guaranteed-valid query from the actual table schema

**Output to state:** `{ "sql_results": SQLAnalysisResult }`

Each result includes: `query_name`, `purpose`, `sql`, `rows`, `row_count`, `execution_status`, `query_validation_warning` (if repaired).

---

### Node 11 — `detect_patterns_node` — DETERMINISTIC

**File:** `backend/app/orchestration/graph.py` (line 214)
**Service:** `backend/app/services/patterns/detector.py`
**Class:** `PatternDetector`
**Method:** `detect_all(df, profile)`

**Input from state:** `state["df"]` (cleaned), `state["profile"]`

**ALL PYTHON/SCIPY (no LLM):**

1. **Trend Detection** (`detect_trends()`): For up to 3 numeric columns, fits linear regression using `scipy.stats.linregress()`. Computes R-squared, p-value, slope. Classifies direction (increasing/decreasing/stable) based on normalized slope and significance. Generates plain-English description with caveats if R-squared < 0.05 or p >= 0.05.

2. **Concentration Patterns** (`detect_concentrations()`): For categorical columns, computes cumulative share. Only classifies as "Pareto" when top ~20% hold ~80% of volume. Otherwise uses "Inventory Concentration" or "Category Concentration".

3. **Anomaly Detection** (`detect_anomalies()`): IQR (3.0x) and Z-score (|z| >= 3.5). Reports row identifier, actual value, boundary value, severity.

4. **Seasonality** (`detect_seasonality()`): If a datetime column exists, groups by month/quarter and tests for periodic variation.

**Output to state:** `{ "patterns": PatternDetectionResult }` with `.trends`, `.concentrations`, `.anomalies`, `.seasonality`

---

### Node 12 — `render_charts_node` — DETERMINISTIC PLOTLY

**File:** `backend/app/orchestration/graph.py` (line 228)
**Service:** `backend/app/services/visualization/chart_generator.py`
**Class:** `ChartGenerator`
**Method:** `generate_all(df, profile, plan, user_query)`

**Input from state:** `state["df"]` (cleaned), `state["profile"]`, `state["plan"]`, `state.get("user_query")`

**Processing — Pure Python/Plotly (no LLM):**

`ChartGenerator` uses `SemanticRoleClassifier` (from `role_classifier.py`) to determine each column's analytical role. Then generates Plotly chart specifications as Python dicts:
- Time series line charts (if datetime column exists)
- Bar charts (categorical x numeric)
- Scatter plots (numeric x numeric with regression line)
- Box plots (distribution by category)
- Histograms (numeric distributions)
- Correlation heatmaps

Each `PlotlyChartSpec` contains: `chart_id`, `title`, `chart_type`, `data` (Plotly traces), `layout` (Plotly layout dict), `takeaway` (data-backed insight text). The frontend renders these with Plotly.js directly.

**Output to state:** `{ "charts": ChartCollection }`

---

### Node 13 — `generate_insights_node` — FOURTH LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 244)
**Agent:** `backend/app/agents/generate_insights.py`
**Class:** `InsightGenerationAgent`
**Method:** `generate(understanding, statistics, sql_results, patterns, quality, revision_critique)` (line 16)

**Input from state:** `state["understanding"]`, `state["statistics"]`, `state["sql_results"]`, `state["patterns"]`, `state["quality"]`, `state.get("revision_critique")`

**Processing:**

All deterministic evidence computed so far is formatted into text:
- Statistical evidence: mean, median, skewness, Pearson r with significance
- SQL evidence: top 3 rows from each successful query (with validation warnings)
- Pattern evidence: trend direction with R-squared and p-values, concentration shares, anomaly descriptions

System prompt enforces strict statistical rules:
- Correlation is NOT causation
- Statistical significance is NOT practical significance (r=0.15 is NOT a strong signal)
- High stock does NOT mean high performance
- Mean vs median skewness must be accurate
- Pareto label only when top ~20% accounts for ~80%
- Prudent language only ("investigate", "review", not "immediately change")

If `revision_critique` is provided (revision loop), it is appended as "CRITICAL AUDIT FEEDBACK FROM PREVIOUS PASS (YOU MUST RESOLVE THESE)."

Calls: `llm_router.complete("generate_insights", messages, response_model=InsightCollection, temperature=0.1)`

**LLM returns:** `InsightCollection` with 3-6 `InsightItem` objects, each containing: `id`, `title`, `finding`, `evidence`, `what_this_means`, `interpretation`, `implication`, `recommendation`, `confidence` (High/Moderate/Low), `confidence_rationale`.

**Output to state:** `{ "insights": InsightCollection }`

---

### Node 14 — `critic_review_node` — FIFTH LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 269)
**Agent:** `backend/app/agents/critic_review.py`
**Class:** `CriticReviewAgent`
**Method:** `review(insights, statistics, sql_results, patterns)` (line 15)

**Input from state:** `state["insights"]`, `state["statistics"]`, `state["sql_results"]`, `state["patterns"]`

**Processing:**

This is the **adversarial critic** agent. It receives BOTH the ground truth evidence AND the generated insights to audit.

System prompt — 10-point evidence validation checklist:
1. Is the claim directly supported by computed evidence?
2. Is the statistic correctly interpreted (mean vs median, skewness direction)?
3. Is the conclusion stronger than evidence warrants?
4. Is correlation confused with causation?
5. Is statistical significance confused with practical significance?
6. Are variable names correctly interpreted?
7. Is the recommendation supported by the finding?
8. Is the sample size sufficient for the stated confidence?
9. Is "Pareto" backed by >=75-80% concentration?
10. Is language clear and free of jargon?

Temperature: `0.0` (most deterministic possible — critic/auditor role).

**LLM returns:** `CriticReviewResult` with: `approved: bool`, `feedback: str`, `unsupported_claims: List[UnsupportedClaim]` (each with `insight_id`, `claim_text`, `reason`, `ground_truth_fact`), `required_corrections: List[str]`, `severity_of_discrepancy: str`.

**Output to state:** `{ "critic_review": review, "critic_approved": review.approved }`

---

### Node 15 — `revise_insights_node` — ROUTING LOGIC (no LLM)

**File:** `backend/app/orchestration/graph.py`
**Function:** `revise_insights_node(state)` (line 291)
**Conditional edge:** `should_continue_revision(state)` (line 325)

**Input from state:** `state["critic_approved"]`, `state["revision_count"]`, `state["critic_review"]`

**Processing — Pure Python (no LLM):**

If `critic_approved = False` AND `revision_count < 2`:
1. Increments `revision_count`
2. Builds structured `revision_critique` string from critic's feedback and `unsupported_claims`
3. Returns state with `revision_critique` and `critic_approved = False`
4. `should_continue_revision()` returns `"generate_insights"` -> routes BACK to Node 13

If `critic_approved = True` OR `revision_count >= 2`:
1. Forces `critic_approved = True`
2. `should_continue_revision()` returns `"generate_report"` -> routes FORWARD to Node 16

**Output to state:** `{ "revision_count": int, "revision_critique": str | None, "critic_approved": bool }`

**Note:** The `InsightRevisionOrchestrator` class in `revise_insights.py` implements a standalone loop-based version of this logic (used separately in `ReportBuilder.build_report_from_dataset()`). Inside the LangGraph pipeline, the **graph node + conditional edge** handle revision routing — not the orchestrator class directly.

---

### Node 16 — `generate_report_node` — SIXTH LLM CALL

**File:** `backend/app/orchestration/graph.py` (line 332)
**Agent:** `backend/app/agents/generate_report.py`
**Class:** `ReportGenerationAgent`
**Method:** `generate(understanding, profile, quality, statistics, sql_results, patterns, charts, insights, cleaning_summary, filename)` (line 64)

**Input from state:** All major artifacts accumulated across nodes 3-15.

**Processing:**

Pre-LLM formatting (deterministic Python):
- Builds quality breakdown table (duplicate count, missing values, invalid values, category normalization, quality score)
- Formats statistical moments with plain-English descriptions
- Formats correlation pairs with significance info
- Formats SQL query results with validation warnings
- Formats patterns (trends with R-squared and p-values, concentrations, anomalies)
- Formats verified insights in structured markdown blocks

System prompt: 10 mandatory reporting guidelines (same statistical rigor as insights — no causation claims, correct skewness, prudent recommendations, no hallucinations, plain English).

Calls: `llm_router.complete("generate_report", messages, response_model=GeneratedReportMarkdown, temperature=0.1)`

**LLM returns:** `GeneratedReportMarkdown` — a Pydantic model with 13 named markdown string fields: `executive_summary_markdown`, `dataset_overview_markdown`, `data_quality_and_validation_markdown`, `key_findings_markdown`, `distribution_analysis_markdown`, `category_analysis_markdown`, `product_analysis_markdown`, `supplier_analysis_markdown`, `relationship_analysis_markdown`, `trend_analysis_markdown`, `recommendations_markdown`, `limitations_markdown`, `suggested_next_analysis_markdown`.

**Post-LLM assembly (deterministic Python):**
- Constructs 13 `ReportSection` objects
- Assembles `full_markdown` string
- Builds `AnalysisReport` Pydantic model embedding ALL artifacts

**Output to state:** `{ "report": AnalysisReport }`

---

### Node 17 — `render_pdf_node` — DETERMINISTIC REPORTLAB

**File:** `backend/app/orchestration/graph.py` (line 354)
**Service:** `backend/app/services/reporting/pdf_exporter.py`
**Class:** `PDFExporter`
**Method:** `generate_pdf(report: AnalysisReport)`

**Input from state:** `state["report"]`, `state.get("cleaned_df")`

**Processing — No LLM:**
1. `ReportBuilder.cache_cleaned_df(dataset_id, cleaned_df)` — stores to memory + disk
2. `PDFExporter.generate_pdf(report)` — generates styled PDF using ReportLab
3. `ReportBuilder.cache_report(report)` — stores to `_cached_reports` dict + writes to `.cache/reports/{dataset_id}.json`

**PDF Generation:**
- `NumberedCanvas` (custom ReportLab canvas at line 19) implements two-pass page numbering
- Color palette: `#3E2723` (dark), `#AD8B73` (primary), `#CEAB93` (accent)
- Renders all 13 report sections as formatted paragraphs, spacers, HR lines, quality breakdown table

**Output to state:** `{ "pdf_bytes": bytes }`

---

## 6. Agent Implementations — Detailed Trace

### Summary of All 6 LLM Calls

| Node | Agent Class | Agent Name (key) | Response Model | Temperature |
|------|-------------|------------------|----------------|-------------|
| 5 | `DatasetUnderstandingAgent` | `understand_dataset` | `DatasetUnderstanding` | 0.1 |
| 6 | `AnalysisPlanningAgent` | `plan_analysis` | `AnalysisPlan` | 0.1 |
| 8 | `SQLGenerationAgent` | `generate_sql` | `SQLGenerationResponse` | 0.1 |
| 13 | `InsightGenerationAgent` | `generate_insights` | `InsightCollection` | 0.1 |
| 14 | `CriticReviewAgent` | `critic_review` | `CriticReviewResult` | 0.0 |
| 16 | `ReportGenerationAgent` | `generate_report` | `GeneratedReportMarkdown` | 0.1 |

**All other nodes (1, 2, 3, 4, 7, 9, 10, 11, 12, 15, 17) are 100% deterministic Python — no LLM.**

---

## 7. LLM Router and Provider Failover

**File:** `backend/app/llm/router.py`
**Class:** `LLMRouter`
**Singleton:** `llm_router = LLMRouter.get_instance()` (line 132)
**Key method:** `complete(agent_name, messages, response_model, temperature)` (line 72)

### What Happens on Every LLM Call

```
Agent calls llm_router.complete(...)
        |
DataPrivacyFilter.redact_text() applied to ALL messages
  (backend/app/llm/privacy.py)
  - Email -> [EMAIL_REDACTED]
  - Phone -> [PHONE_REDACTED]
  - SSN   -> [SSN_REDACTED]
  - Card  -> [CARD_REDACTED]
  - Keys  -> [SECRET_REDACTED]
        |
Build provider list (priority order):
  1. Groq keys (GROQ_API_KEY_1, _2, _3) — those not in cooldown
  2. Gemini keys (GEMINI_API_KEY_1, _2) — those not in cooldown
        |
If no providers available:
  -> MockLLMProvider.generate_structured()  <- deterministic fallback
        |
Try each provider in order:
  GroqProvider.generate_structured()
    (backend/app/llm/groq_provider.py)
    - Injects Pydantic JSON schema into system prompt
    - Calls groq SDK: client.chat.completions.create(response_format={"type": "json_object"})
    - Validates: response_model.model_validate_json(content)
    - On rate limit (429/quota/limit): mark_cooldown(key_id) for 60 seconds, try next
        |
  GeminiProvider.generate_structured()
    (backend/app/llm/gemini_provider.py)
    - Uses google-genai SDK
    - Sets response_mime_type="application/json", response_schema=response_model
    - Calls client.models.generate_content(...)
    - Validates: response_model.model_validate_json(raw_text)
        |
If ALL providers fail:
  -> MockLLMProvider.generate_structured()  <- graceful fallback with pre-built responses
```

### Cooldown Mechanism

When a provider returns a rate limit error (detected by checking for "rate", "429", "quota", "limit" in the error string), `mark_cooldown(key_id)` sets `cooldowns[key_id] = time.time() + 60.0`. The `is_cooling_down(key_id)` check skips that key for 60 seconds on subsequent calls.

---

## 8. State Flow Throughout the Pipeline

**File:** `backend/app/orchestration/state.py`
**TypedDict:** `AnalysisWorkflowState`

### State Field Reference Table

| Field | Type | Set by Node | Read by Node |
|-------|------|-------------|--------------|
| `dataset_id` | str | initial + 2 | All |
| `filename` | str | 1 | 5, 4, 16 |
| `file_bytes` | bytes | initial | 1, 2 |
| `file_extension` | str | 1 | 2 |
| `table_name` | str | 2 (raw), 4 (cleaned) | 3, 8, 9, 10 |
| `raw_table_name` | str | 2 | — |
| `cleaned_table_name` | str | 4 | — |
| `df` | pd.DataFrame | 2 (raw), 4 (cleaned) | 3, 7, 11, 12 |
| `raw_df` | pd.DataFrame | 2 | 4 |
| `cleaned_df` | pd.DataFrame | 4 | 17 |
| `cleaning_summary` | dict | 4 | 16 |
| `profile` | DatasetProfile | 3, 4 (re-profile) | 5, 6, 7, 8, 11, 12, 16 |
| `quality` | QualityReport | 3 | 5, 13, 16 |
| `understanding` | DatasetUnderstanding | 5 | 6, 13, 16 |
| `plan` | AnalysisPlan | 6 | 7, 8, 12 |
| `statistics` | StatisticalAnalysisResult | 7 | 13, 14, 16 |
| `sql_queries` | List[GeneratedSQLQuery] | 8 | 9, 10 |
| `validated_sql_queries` | List[GeneratedSQLQuery] | 9 | 10 |
| `sql_results` | SQLAnalysisResult | 10 | 13, 14, 16 |
| `patterns` | PatternDetectionResult | 11 | 13, 14, 16 |
| `charts` | ChartCollection | 12 | 16 |
| `insights` | InsightCollection | 13 | 14, 16 |
| `critic_review` | CriticReviewResult | 14 | 15 |
| `critic_approved` | bool | 14, 15 | conditional edge |
| `revision_count` | int | initial=0, 15 | conditional edge |
| `revision_critique` | str | 15 | 13 (revision loop) |
| `report` | AnalysisReport | 16 | 17 |
| `pdf_bytes` | bytes | 17 | — |

### State Evolution

```
Initial: { dataset_id, filename, file_bytes, logs: [], revision_count: 0, critic_approved: False }
  | Node 1 adds: file_extension
  | Node 2 adds: table_name (raw), raw_table_name, df (raw), raw_df
  | Node 3 adds: profile (raw), quality
  | Node 4 adds: df (cleaned), cleaned_df, table_name (cleaned), cleaning_summary, profile (cleaned)
  | Node 5 adds: understanding  [LLM]
  | Node 6 adds: plan  [LLM]
  | Node 7 adds: statistics
  | Node 8 adds: sql_queries  [LLM]
  | Node 9 adds: validated_sql_queries
  | Node 10 adds: sql_results
  | Node 11 adds: patterns
  | Node 12 adds: charts
  | Node 13 adds: insights  [LLM]
  | Node 14 adds: critic_review, critic_approved  [LLM]
  | Node 15 adds: revision_count, revision_critique (may loop back to Node 13)
  | Node 16 adds: report  [LLM]
  | Node 17 adds: pdf_bytes
Final state contains all the above fields populated.
```

---

## 9. SSE Streaming to the Frontend

### Backend SSE Format

**File:** `backend/app/orchestration/events.py`
**Function:** `format_sse_event(event_type, data)` (line 45)

Each SSE message:
```
event: <event_type>
data: <json_payload_with_timestamp>

```
(two newlines at end)

`clean_nan_and_inf()` recursively replaces float `NaN` and `Infinity` with `None` before JSON serialization.

### Events Emitted During Pipeline

| Event Type | When Emitted | Key Data |
|------------|-------------|----------|
| `pipeline_start` | Before Node 1 | `dataset_id`, `filename`, `total_steps: 17` |
| `step_complete` | After each node | `step`, `step_index`, `label`, `agent`, `preview`, `status: "completed"` |
| `complete` | After Node 17 | `dataset_id`, `filename`, `report` (full JSON), `status: "success"` |
| `error` | On exception | `dataset_id`, `error`, `step` |

### Live Preview Snippets (per step_complete event)

| Node Name | Preview Content |
|-----------|----------------|
| `profile_and_audit` | `{ quality_score, grade }` |
| `understand_dataset` | `{ domain, kpis: [top 3 names] }` |
| `run_statistical_analysis` | `{ metrics_count, correlations_count }` |
| `generate_insights` | `{ insights_count, top_title }` |
| `generate_report` | `{ report_title, sections_count }` |

### Frontend SSE Parsing

**File:** `frontend/src/App.jsx`
**Function:** `handleStartAnalysis()` (line 198)

```javascript
const response = await fetch(`${apiBase}/api/analyze/stream`, {
  method: 'POST', body: formData, signal: abortControllerRef.current.signal
});
const reader = response.body.getReader();
const decoder = new TextDecoder('utf-8');
let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  // Split on double newline (SSE event boundary)
  const lines = buffer.split('\n\n');
  buffer = lines.pop();
  for (const block of lines) {
    // Extract event: and data: lines
    // Parse JSON from data line
    // Dispatch based on eventType:
    //   step_complete   -> setCompletedSteps(), setLivePreviews()
    //   pipeline_complete -> setFinalReport(), setStage('dashboard')
    //   pipeline_error  -> setErrorMsg(), setStage('upload')
  }
}
```

> **Implementation note:** The backend emits event type `"complete"` (line 74 in `analysis.py`) but the frontend checks for `"pipeline_complete"` (line 271 in `App.jsx`). These do not match. After the SSE stream ends naturally (`reader.done = true`), the while loop exits and the final `complete` event's data (which includes the report) has been received but may not have triggered `setFinalReport()` through the `pipeline_complete` branch. The pipeline functions because the report is also cached server-side and retrievable via `GET /api/dataset/{id}/report`.

---

## 10. Visualization Flow

```
Node 12: render_charts_node
        |
ChartGenerator.generate_all(df, profile, plan, user_query)
  backend/app/services/visualization/chart_generator.py
        |
SemanticRoleClassifier.classify_columns()
  backend/app/services/visualization/role_classifier.py
  -> determines each column's analytical role (time dimension, metric, category, etc.)
        |
Generate PlotlyChartSpec dicts (pure Python/Plotly data structures):
  - generate_time_series_chart()   <- if datetime column exists
  - generate_bar_chart()           <- categorical x numeric group-by
  - generate_scatter_chart()       <- numeric x numeric
  - generate_box_plot()            <- distribution by category
  - generate_histogram()           <- numeric frequency distribution
        |
Each PlotlyChartSpec contains:
  { chart_id, title, chart_type,
    data: [Plotly trace dicts],
    layout: { Plotly layout dict },
    takeaway: "data-backed plain text finding" }
        |
ChartCollection stored in state["charts"]
        |
Node 16 embeds ChartCollection into AnalysisReport
        |
Final SSE "complete" event carries AnalysisReport JSON (includes charts)
        |
Frontend: VisualizationsTab.jsx renders tab
  uses ChartRenderer.jsx for each chart
  Plotly.js renders from data/layout spec
```

**Color palette in all charts:**
- Primary: `#AD8B73`
- Dark: `#3E2723`
- Accent: `#CEAB93`
- Background: transparent (`rgba(0,0,0,0)`)
- Font: "Plus Jakarta Sans, Inter, sans-serif"

No charts are rendered as images server-side. The Plotly spec (JSON) is sent to the frontend and rendered in the browser.

---

## 11. PDF Export Flow

### Automatic PDF (Node 17 in pipeline)

```
Node 17: render_pdf_node
  backend/app/orchestration/graph.py (line 354)
        |
PDFExporter.generate_pdf(report: AnalysisReport)
  backend/app/services/reporting/pdf_exporter.py
        |
io.BytesIO buffer <- in-memory PDF
SimpleDocTemplate from reportlab (letter page size)
        |
Render elements:
  - Paragraphs (ParagraphStyle objects for title, heading, body, footer)
  - Spacers and HRFlowables (separator lines)
  - Table for quality breakdown (TableStyle with color palette)
  - KeepTogether blocks for each section
        |
NumberedCanvas (line 19) - two-pass rendering:
  showPage() saves state per page
  save() computes total pages, renders "Page X of Y" on each page
        |
pdf_bytes returned
        |
ReportBuilder.cache_report(report)
  -> memory: _cached_reports[dataset_id] = report
  -> disk: .cache/reports/{dataset_id}.json
        |
pdf_bytes stored in state["pdf_bytes"]
```

### On-Demand PDF Download

**Route:** `GET /api/dataset/{dataset_id}/report/pdf`
**File:** `backend/app/api/routes/report.py`
**Function:** `download_pdf_report(dataset_id)` (line 106)

```
Frontend "Download PDF" button click
        |
GET /api/dataset/{dataset_id}/report/pdf
        |
ReportBuilder.get_report(dataset_id)  <- check memory cache, then disk .cache/reports/
        |
If not cached: rebuild from DuckDB table
        |
PDFExporter.generate_pdf(report)
        |
Response(content=pdf_bytes, media_type="application/pdf",
  headers={"Content-Disposition": "attachment; filename=analysis_report_{id}.pdf"})
        |
Browser downloads the file
```

---

## 12. Error Handling Flow

### File Upload Errors (Path A — `/api/upload`)

**File:** `backend/app/api/routes/upload.py` (lines 75-83)

```
FileValidationError (unsupported format, too large, empty)
  -> HTTP 400 with e.message

IngestionError (CSV parse failure, empty DataFrame, exceeded row/col limits)
  -> HTTP 422 with e.message

Any other Exception
  -> HTTP 500 "Internal processing error while ingesting dataset."
```

### Pipeline Errors (Path B — SSE)

**File:** `backend/app/api/routes/analysis.py` (lines 85-91)

```
Exception inside a LangGraph node
  -> event_generator_from_graph() catches it
  -> Emits SSE: event="error", data={dataset_id, error: str(e), step: current_state.get("current_step")}
  -> Frontend App.jsx (line 293): eventType === 'pipeline_error'
     -> setErrorMsg(error_message), setStage('upload')
```

### SQL Security Errors

**File:** `backend/app/services/sql/validator.py`

```
Disallowed keyword detected (e.g., DROP, INSERT)
  -> validate_sql() returns (False, "Forbidden SQL keyword detected: 'DROP'", "")
  -> validate_sql_node() in graph.py silently drops the query with logger.warning
  -> Pipeline continues with remaining valid queries
```

### SQL Execution Errors

**File:** `backend/app/services/sql/executor.py`

```
DuckDB raises exception
  -> repair_query_syntax() attempts to fix common LLM mistakes
  -> If still fails: synthesize_fallback_query() generates guaranteed-valid query
  -> If fallback also fails: result.execution_status = "error", error_message logged
  -> Pipeline continues with remaining queries
```

### LLM Provider Errors

**File:** `backend/app/llm/router.py`

```
Rate limit / quota / auth failure (429 / "rate" / "quota" / "limit" in error)
  -> mark_cooldown(key_id) for 60 seconds
  -> Try next provider in pool

All configured external providers fail
  -> logger.warning("All external providers failed. Falling back to deterministic mock generator.")
  -> MockLLMProvider.generate_structured() returns pre-built Pydantic objects
  -> Pipeline continues with mock data (never crashes)
```

### Insight Revision Failure (after max revisions)

**File:** `backend/app/agents/revise_insights.py` (line 89-102)

When `InsightRevisionOrchestrator` is used (in `ReportBuilder.build_report_from_dataset()`):
```
After MAX_REVISIONS (2) exceeded and still unapproved:
  -> For each flagged insight: confidence downgraded to "Caveat"
  -> finding appended with "(Note: Claim subject to statistical data limitation caveat)."
  -> overall_confidence_rating set to "Caveat"
  -> Pipeline continues (never blocks)
```

### Global FastAPI Exception Handler

**File:** `backend/app/main.py` (lines 73-79)

```python
@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    -> HTTP 500 { "error": "InternalServerError", "message": "An unexpected error occurred." }
```

### User Cancellation

**File:** `frontend/src/App.jsx`
**Function:** `handleStopAnalysis()` (line 322)

```
User clicks "Stop"
  -> abortControllerRef.current.abort()
  -> streamReaderRef.current.cancel()
  -> fetch() throws AbortError
  -> Caught: err.name === 'AbortError'
  -> setStatusNotice('Analysis stream was cancelled by user.')
  -> setStage('upload'), reset all progress state
```

---

## 13. File-by-File Connection Map

```
frontend/src/main.jsx
  (mounts React app)
        |
frontend/src/App.jsx
  (reads API URL from)
        |
frontend/src/services/apiConfig.js
  (POST /api/analyze/stream with FormData)
        |
backend/app/api/routes/analysis.py
  analyze_file_stream()
  -> constructs initial_state
  -> returns StreamingResponse wrapping
        |
backend/app/api/routes/analysis.py
  event_generator_from_graph()
  -> calls .astream() on
        |
backend/app/orchestration/graph.py
  analysis_workflow_graph (compiled StateGraph)
  -> reads state from
        |
backend/app/orchestration/state.py
  AnalysisWorkflowState TypedDict
  -> emits events using
        |
backend/app/orchestration/events.py
  STEP_METADATA, format_sse_event()

=== NODE 1 ===
backend/app/services/ingestion/validator.py
  FileValidator.validate_file_metadata()

=== NODE 2 ===
backend/app/services/ingestion/loader.py
  DatasetLoader.load_and_sanitize()
        |
backend/app/services/ingestion/duckdb_manager.py
  DuckDBManager.register_dataframe()

=== NODE 3 ===
backend/app/services/profiling/profiler.py
  DatasetProfiler.profile_dataset()
        |
backend/app/services/quality/checker.py
  QualityChecker.audit_dataset()

=== NODE 4 ===
backend/app/services/cleaning/cleaner.py
  DataCleaner.clean_dataset()
        |
backend/app/services/reporting/report_builder.py
  ReportBuilder.cache_cleaned_df()

=== NODE 5 [LLM] ===
backend/app/agents/understand_dataset.py
  DatasetUnderstandingAgent.analyze()
        |
backend/app/llm/router.py
  LLMRouter.complete()
        |
backend/app/llm/privacy.py
  DataPrivacyFilter.redact_text()
        |
backend/app/llm/groq_provider.py  OR
backend/app/llm/gemini_provider.py  OR
backend/app/llm/mock_provider.py

=== NODE 6 [LLM] ===
backend/app/agents/plan_analysis.py
  AnalysisPlanningAgent.plan()
  -> calls llm_router.complete()
  -> calls validate_and_clean_plan() [deterministic column validation]

=== NODE 7 ===
backend/app/services/statistics/engine.py
  StatisticalEngine.run_analysis()
  -> compute_univariate_metrics() [NumPy/Pandas]
  -> compute_correlations() [scipy.stats.pearsonr, spearmanr]
  -> compute_groupby_analyses() [pandas groupby + scipy.stats.f_oneway]

=== NODE 8 [LLM] ===
backend/app/agents/generate_sql.py
  SQLGenerationAgent.generate()
  -> calls llm_router.complete()

=== NODE 9 ===
backend/app/services/sql/validator.py
  SQLValidator.validate_sql()

=== NODE 10 ===
backend/app/services/sql/executor.py
  SQLExecutor.execute_queries()
        |
backend/app/services/ingestion/duckdb_manager.py
  duckdb_manager.execute_query()

=== NODE 11 ===
backend/app/services/patterns/detector.py
  PatternDetector.detect_all()
  -> detect_trends() [scipy.stats.linregress]
  -> detect_concentrations() [cumulative share]
  -> detect_anomalies() [IQR, Z-score]
  -> detect_seasonality()

=== NODE 12 ===
backend/app/services/visualization/chart_generator.py
  ChartGenerator.generate_all()
        |
backend/app/services/visualization/role_classifier.py
  SemanticRoleClassifier

=== NODE 13 [LLM] ===
backend/app/agents/generate_insights.py
  InsightGenerationAgent.generate()
  -> calls llm_router.complete()

=== NODE 14 [LLM, temperature=0.0] ===
backend/app/agents/critic_review.py
  CriticReviewAgent.review()
  -> calls llm_router.complete()

=== NODE 15 (conditional routing, no LLM) ===
backend/app/orchestration/graph.py
  revise_insights_node() + should_continue_revision()
  IF unapproved + revisions < 2: LOOP BACK to Node 13
  ELSE: PROCEED to Node 16

=== NODE 16 [LLM] ===
backend/app/agents/generate_report.py
  ReportGenerationAgent.generate()
  -> calls llm_router.complete()
  -> assembles AnalysisReport with all artifacts

=== NODE 17 ===
backend/app/services/reporting/pdf_exporter.py
  PDFExporter.generate_pdf()
        |
backend/app/services/reporting/report_builder.py
  ReportBuilder.cache_report()

Final SSE "complete" event -> frontend App.jsx
        |
frontend/src/components/OverviewTab.jsx
frontend/src/components/DataQualityTab.jsx
frontend/src/components/StatisticsTab.jsx
frontend/src/components/VisualizationsTab.jsx  (uses ChartRenderer.jsx + Plotly.js)
frontend/src/components/InsightsTab.jsx
frontend/src/components/ReportMarkdownTab.jsx
frontend/src/components/CleanDataTab.jsx
```

---

## 14. Master Execution Flow Diagram

```
APPLICATION START
  backend/app/main.py                         <- FastAPI app, CORS, exception handlers
  backend/app/services/ingestion/duckdb_manager.py  <- in-memory DuckDB singleton
  backend/app/llm/router.py                   <- LLMRouter singleton with provider pool
  |
  frontend/src/main.jsx                       <- React mount
  frontend/src/App.jsx                        <- stage='upload', load history
  |
USER SELECTS FILE
  App.jsx: setSelectedFile(file)
  |
USER CLICKS "ANALYZE"
  App.jsx: handleStartAnalysis() (line 198)
  frontend/src/services/apiConfig.js: getApiBaseUrl()
  |
POST /api/analyze/stream (multipart file)
  backend/app/api/routes/analysis.py: analyze_file_stream()
    -> initial_state = { dataset_id, filename, file_bytes, logs:[], revision_count:0 }
    -> StreamingResponse -> event_generator_from_graph(initial_state)
  |
SSE: pipeline_start -> Frontend: PipelineTracker.jsx shows 17 steps
  |
backend/app/orchestration/graph.py: analysis_workflow_graph.astream(state)
  |
==============================================================
NODE 1: validate_file_node
  backend/app/services/ingestion/validator.py
  FileValidator.validate_file_metadata()
  -> sanitized_name, file_extension
  SSE: step_complete { step: "validate_file", step_index: 1 }
==============================================================
NODE 2: load_dataset_node
  backend/app/services/ingestion/loader.py
  DatasetLoader.load_and_sanitize() -> pd.DataFrame
  backend/app/services/ingestion/duckdb_manager.py
  duckdb_manager.register_dataframe() -> raw DuckDB table "raw_dataset_<uuid>"
  SSE: step_complete { step: "load_dataset", step_index: 2 }
==============================================================
NODE 3: profile_and_audit_node
  backend/app/services/profiling/profiler.py
  DatasetProfiler.profile_dataset() -> DatasetProfile (column types, stats, nulls)
  backend/app/services/quality/checker.py
  QualityChecker.audit_dataset() -> QualityReport (score 0-100, grade, issues)
  SSE: step_complete { step: "profile_and_audit", preview: {quality_score, grade} }
==============================================================
NODE 4: clean_and_standardize_node
  backend/app/services/cleaning/cleaner.py
  DataCleaner.clean_dataset() -> cleaned_df + CleaningSummary
  backend/app/services/reporting/report_builder.py
  ReportBuilder.cache_cleaned_df() -> .cache/cleaned/{id}.csv
  duckdb_manager.register_dataframe() -> cleaned DuckDB table "dataset_<uuid>"
  DatasetProfiler.profile_dataset() -> updated profile (on cleaned data)
  SSE: step_complete { step: "clean_and_standardize", step_index: 4 }
==============================================================
NODE 5: understand_dataset_node  [LLM CALL #1]
  backend/app/agents/understand_dataset.py
  DatasetUnderstandingAgent.analyze(profile, quality, filename)
  backend/app/llm/router.py
  llm_router.complete("understand_dataset", messages, DatasetUnderstanding, 0.1)
  -> Groq llama-3.3-70b OR Gemini 2.5-flash OR MockProvider
  -> DatasetUnderstanding (domain, summary, KPIs, business questions)
  SSE: step_complete { step: "understand_dataset", preview: {domain, kpis} }
==============================================================
NODE 6: plan_analysis_node  [LLM CALL #2]
  backend/app/agents/plan_analysis.py
  AnalysisPlanningAgent.plan(profile, understanding)
  -> LLM generates raw AnalysisPlan
  -> validate_and_clean_plan() removes any hallucinated column names
  -> validated AnalysisPlan (group_by_analyses, sql_query_goals, correlation_pairs)
  SSE: step_complete { step: "plan_analysis", step_index: 6 }
==============================================================
NODE 7: run_statistical_analysis_node  [DETERMINISTIC Python/SciPy]
  backend/app/services/statistics/engine.py
  StatisticalEngine.run_analysis(df, profile, plan)
  -> compute_univariate_metrics() [mean, median, std, IQR, skewness via NumPy/Pandas]
  -> compute_correlations() [scipy.stats.pearsonr, spearmanr]
  -> compute_groupby_analyses() [pandas groupby + scipy.stats.f_oneway ANOVA]
  -> StatisticalAnalysisResult
  SSE: step_complete { step: "run_statistical_analysis", preview: {metrics_count, correlations_count} }
==============================================================
NODE 8: generate_sql_node  [LLM CALL #3]
  backend/app/agents/generate_sql.py
  SQLGenerationAgent.generate(profile, plan, table_name)
  -> LLM generates DuckDB SQL queries from sql_query_goals
  -> List[GeneratedSQLQuery]
  SSE: step_complete { step: "generate_sql", step_index: 8 }
==============================================================
NODE 9: validate_sql_node  [DETERMINISTIC Security]
  backend/app/services/sql/validator.py
  SQLValidator.validate_sql(sql, expected_table)
  -> Regex scan for disallowed keywords, functions, URL schemes
  -> Enforces SELECT/WITH only
  -> Unsafe queries dropped silently
  SSE: step_complete { step: "validate_sql", step_index: 9 }
==============================================================
NODE 10: execute_sql_node  [DETERMINISTIC DuckDB]
  backend/app/services/sql/executor.py
  SQLExecutor.execute_queries(queries, dataset_id, table_name)
  -> repair_query_syntax() for LLM errors
  -> duckdb_manager.execute_query(sql)
  -> On failure: synthesize_fallback_query()
  -> SQLAnalysisResult (rows per query)
  SSE: step_complete { step: "execute_sql", step_index: 10 }
==============================================================
NODE 11: detect_patterns_node  [DETERMINISTIC Python/SciPy]
  backend/app/services/patterns/detector.py
  PatternDetector.detect_all(df, profile)
  -> detect_trends() [scipy.stats.linregress, R-squared, p-value]
  -> detect_concentrations() [cumulative share, Pareto check]
  -> detect_anomalies() [IQR 3.0x, Z-score >=3.5]
  -> detect_seasonality() [monthly/quarterly grouping]
  SSE: step_complete { step: "detect_patterns", step_index: 11 }
==============================================================
NODE 12: render_charts_node  [DETERMINISTIC Plotly]
  backend/app/services/visualization/chart_generator.py
  ChartGenerator.generate_all(df, profile, plan, user_query)
  backend/app/services/visualization/role_classifier.py
  SemanticRoleClassifier -> column role detection
  -> Time series, bar, scatter, box, histogram chart specs
  -> ChartCollection (List[PlotlyChartSpec with data + layout dicts])
  SSE: step_complete { step: "render_charts", step_index: 12 }
==============================================================
NODE 13: generate_insights_node  [LLM CALL #4]
  backend/app/agents/generate_insights.py
  InsightGenerationAgent.generate(understanding, statistics, sql_results,
                                   patterns, quality, revision_critique)
  -> Formats all deterministic evidence as structured text prompt
  -> Enforces strict statistical rules in system prompt
  -> InsightCollection (3-6 evidence-grounded insights)
  SSE: step_complete { step: "generate_insights", preview: {insights_count, top_title} }
==============================================================
NODE 14: critic_review_node  [LLM CALL #5, temperature=0.0]
  backend/app/agents/critic_review.py
  CriticReviewAgent.review(insights, statistics, sql_results, patterns)
  -> Sends both ground-truth evidence AND insights to LLM
  -> 10-point audit checklist in system prompt
  -> CriticReviewResult (approved: bool, unsupported_claims, feedback)
  SSE: step_complete { step: "critic_review", step_index: 14 }
==============================================================
NODE 15: revise_insights_node  [ROUTING LOGIC, no LLM]
  backend/app/orchestration/graph.py: revise_insights_node()
  should_continue_revision():
    IF critic_approved=False AND revision_count < 2:
      revision_count++, build revision_critique string
      ROUTE BACK TO NODE 13 (loop, max 2 iterations)
    ELSE:
      ROUTE TO NODE 16
  SSE: step_complete { step: "revise_insights", step_index: 15 }
==============================================================
NODE 16: generate_report_node  [LLM CALL #6]
  backend/app/agents/generate_report.py
  ReportGenerationAgent.generate(understanding, profile, quality, statistics,
                                  sql_results, patterns, charts, insights,
                                  cleaning_summary, filename)
  -> Pre-formats all evidence deterministically
  -> LLM generates GeneratedReportMarkdown (13 named markdown sections)
  -> Assembles 13 ReportSection objects + full_markdown string
  -> Assembles AnalysisReport Pydantic model (all artifacts embedded)
  SSE: step_complete { step: "generate_report", preview: {report_title, sections_count} }
==============================================================
NODE 17: render_pdf_node  [DETERMINISTIC ReportLab]
  backend/app/services/reporting/pdf_exporter.py
  PDFExporter.generate_pdf(report) -> pdf_bytes
  backend/app/services/reporting/report_builder.py
  ReportBuilder.cache_report(report) -> .cache/reports/{dataset_id}.json
  SSE: step_complete { step: "render_pdf", step_index: 17 }
==============================================================
SSE: complete event { dataset_id, filename, report: <full AnalysisReport JSON> }
  |
frontend/src/App.jsx
  SSE stream ends -> while loop exits
  setFinalReport(), setStage('dashboard'), setHistory(updated)
  |
Dashboard renders:
  OverviewTab.jsx      <- dataset summary, quality score card, key metrics
  DataQualityTab.jsx   <- quality score, issue list, breakdown
  StatisticsTab.jsx    <- distributions, correlations, group analyses
  VisualizationsTab.jsx <- Plotly charts from ChartCollection
  InsightsTab.jsx      <- 3-6 evidence-grounded insights with confidence
  ReportMarkdownTab.jsx <- full 13-section markdown report
  CleanDataTab.jsx     <- cleaned dataset preview + CSV/Excel/PDF download
```

---

## 15. How to Explain This Project in an Interview

### What the Project Does

DataPilot is a multi-agent AI data analysis assistant named after a project called "DataPilot." A user uploads a CSV or Excel file, and the system automatically runs a 17-step analytical pipeline combining deterministic statistical computation with LLM-generated intelligence to produce a comprehensive 13-section business intelligence report, interactive Plotly charts, and a downloadable PDF — all without writing a single line of analysis code.

### How the User Interacts with It

1. The user opens the React single-page application (Vite + React, ~952 lines in `App.jsx`).
2. They drag-and-drop a CSV or Excel file, or select a built-in sample dataset.
3. They click "Analyze." The frontend sends the file to the backend as a multipart POST to `/api/analyze/stream`.
4. The backend immediately streams progress updates via Server-Sent Events (SSE). The frontend shows a live pipeline tracker with 17 steps ticking off as each completes, with live preview cards showing quality score, domain, stats count, etc.
5. When all 17 steps finish, the frontend transitions to a full dashboard with 7 tabs: Overview, Data Quality, Statistics, Visualizations, Insights, Full Report, and Clean Data.
6. The user can download a PDF report or export the cleaned dataset as CSV or Excel.

### What Happens After Dataset Upload

The file bytes are passed as initial state into a **LangGraph `StateGraph`** with 17 nodes. The first 4 nodes are purely deterministic:
1. **File validation** — format, size, encoding
2. **Dataset loading** — pandas DataFrame + in-memory DuckDB table
3. **Profiling and quality audit** — per-column statistics, data types, quality score 0-100
4. **Data cleaning** — imputation, deduplication, normalization, re-profiling

From Node 4 onward, all analysis operates on the cleaned data.

### How the Data Flows

After cleaning, the data exists in two places simultaneously:
- A **pandas DataFrame** in the `AnalysisWorkflowState` TypedDict (passed between LangGraph nodes)
- An **in-memory DuckDB table** (accessed by SQL query nodes)

The cleaned DataFrame is also persisted to disk in `.cache/cleaned/` for later retrieval. The final `AnalysisReport` is persisted to `.cache/reports/` as JSON.

### How the Multi-Agent System Works

The project has **6 LLM-calling agents** and **11 deterministic processing nodes**:

**LLM Agents:**
1. `DatasetUnderstandingAgent` (Node 5) — Infers domain, KPIs, and business questions from column metadata
2. `AnalysisPlanningAgent` (Node 6) — Plans which statistics to compute, SQL queries to run, columns to correlate — with post-LLM column validation
3. `SQLGenerationAgent` (Node 8) — Generates DuckDB SQL queries targeting the cleaned table
4. `InsightGenerationAgent` (Node 13) — Translates computed evidence into 3-6 business insights with strict statistical rules
5. `CriticReviewAgent` (Node 14) — Adversarially audits insights against ground-truth evidence (10-point checklist, temperature=0.0)
6. `ReportGenerationAgent` (Node 16) — Compiles all artifacts into a 13-section executive report

**Deterministic Nodes:** Nodes 1, 2, 3, 4, 7, 9, 10, 11, 12, 15, 17 use only Python, NumPy, SciPy, Pandas, DuckDB, Plotly, and ReportLab.

### How Agents Communicate

Agents do **NOT** communicate directly. They communicate through the shared `AnalysisWorkflowState` TypedDict managed by LangGraph. Each node reads from state what it needs and writes its output back. The next node receives the updated state automatically.

### How the LLM Is Actually Used

The LLM (`llama-3.3-70b-versatile` via Groq, or `gemini-2.5-flash` via Google, or a mock fallback) is invoked exactly 6 times. Each call goes through:
1. **Privacy filtering:** `DataPrivacyFilter.redact_text()` strips emails, phones, SSNs, credit cards from all prompt content before sending to any external API
2. **Provider failover:** `LLMRouter.complete()` tries Groq keys first, then Gemini keys. Rate-limited keys are put on 60-second cooldown. If all providers fail, `MockLLMProvider` returns pre-built Pydantic objects
3. **Structured output:** All LLM calls use JSON mode / response schema. The LLM returns JSON that is validated by `response_model.model_validate_json()`

**The key design principle:** The LLM only generates TEXT (interpretations, SQL queries, insights, report prose). All NUMBERS come from deterministic Python computations. This makes the system statistically trustworthy — the LLM can only interpret what the code has already calculated.

### The Revision Loop (Critic-Insight Feedback)

After insights are generated (Node 13), the `CriticReviewAgent` (Node 14) audits them against the computed evidence using a 10-point checklist. If the critic rejects them, the `revise_insights_node` (Node 15) packages the critic's feedback into a `revision_critique` string and routes execution BACK to `generate_insights_node`. This feedback loop can occur at most **twice** (enforced by `revision_count <= 2` check). After 2 failed revisions, disputed insights are marked with "Caveat" confidence instead of blocking the pipeline.

### How the Final Result Is Produced

After all 17 nodes complete:
1. The `AnalysisReport` Pydantic model embeds all artifacts: profile, quality, statistics, SQL results, patterns, charts (as Plotly specs), insights, cleaning summary, data quality breakdown table, 13 markdown sections
2. A PDF is rendered server-side using ReportLab with a custom color palette
3. Both report JSON and cleaned CSV are cached to disk for later retrieval
4. The final SSE event carries the full report JSON to the frontend
5. The frontend renders the full dashboard from the received JSON — no additional API calls needed for the main display

**In summary:** This project demonstrates a production-grade multi-agent data analysis pipeline where LLM intelligence (domain understanding, planning, SQL generation, insights, critique, reporting) is grounded by and constrained to deterministic statistical computation (profiling, cleaning, statistics, SQL execution, pattern detection, visualization), with a feedback loop ensuring analytical correctness, streamed in real-time to a React dashboard.

---

*This document was generated by direct inspection of the actual source code. Every file path, class name, function name, and behavior described here has been confirmed from the codebase. Nothing has been assumed or invented.*
