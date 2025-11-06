# Adaptive Insight Generation (AIG) Architecture Framework Documentation

**Version:** 1.0  
**Date:** 2025-11-06  
**Next Review:** Q1 2026  
**Maintainers:** Data Platform Team / AI Insights Team  

---

## 1. Overview and Core Philosophy

The **Adaptive Insight Generation (AIG)** framework defines a modern, explainable, and efficient data + AI architecture for generating pre-analytics from both transactional and non-transactional data sources, and serving grounded insights via a **Retrieval-Augmented Generation (RAG)** system.

The guiding principle is to **separate expensive batch computation (Pre-Analytics)** from **real-time insight synthesis (RAG)** — optimizing cost, latency, and reliability while preserving **traceability and factual grounding**.

---

## 2. Architecture Diagram

### Adaptive Insight Generation (AIG) — Text-Based Architecture Flow

```
[ DATA SOURCES ]
  ├─ Transactional: royalties, invoices, plays, logs
  ├─ Non-Transactional: rightsholder, metadata, dimensions
  └─ External: currency, DSP rates, benchmarks
        │
        ▼
[ DATA LAKEHOUSE ]
  (Parquet / Iceberg / Delta on GCS, S3, or ADLS)
  ├─ BRONZE: raw ingestion
  ├─ SILVER: cleaned facts & standardized dimensions
  ├─ GOLD: curated analytics marts (aggregated facts)
  └─ Outputs structured JSON + text "insight chunks"
        │
        ▼
[ INSIGHT GENERATION & CURATION (Pre-Analytics Engine) ]
  ├─ Metric computation (dbt / Spark / Trino / DuckDB)
  ├─ Trend & anomaly detection
  ├─ Narrative & validation engine (LLM + rules)
  └─ Embedding + Registrar → writes to RAG index
        │
        ▼
[ INSIGHT STORAGE + RAG INDEX ]
  ├─ Vector database: pgvector / Pinecone / Weaviate / Chroma
  ├─ Each record: {embedding, summary_text, metadata: (entity, period, metric)}
  └─ Metadata store: Postgres / Elastic / Bigtable
        │
        ▼
[ INSIGHT API / APPLICATION LAYER ]
  ├─ REST / GraphQL API (FastAPI, Node, Go)
  ├─ Retrieves top-k insights via hybrid RAG search
  ├─ Applies metadata filters (period, region, rightsholder, etc.)
  ├─ Calls small LLM for reasoning & narrative synthesis
  └─ Returns grounded explanations + source references
        │
        ▼
[ USER INTERFACE / DASHBOARD / BOT ]
  ├─ Web UI, Slack, or chat interface
  ├─ Query templates or controlled natural language
  └─ Feedback capture ("Not relevant" → logs gap)
        │
        ▼
[ FEEDBACK + AUTO-LEARNING LOOP ]
  ├─ Logs unanswered / low-confidence queries
  ├─ Detects missing metrics or entities
  └─ Triggers new pre-analytics computation for coverage expansion
```

*This ASCII-style diagram captures the data and insight flow of the AIG architecture without requiring any images.*

---

### **2. Data Lakehouse**
- **Storage:** Parquet / Iceberg / Delta on GCS, S3, or ADLS  
- **Layers:**
  - **BRONZE:** Raw, immutable ingestion  
  - **SILVER:** Cleaned, standardized facts and dimensions  
  - **GOLD:** Curated analytics marts (highly aggregated and trusted facts)
- **Output:** Structured `GOLD` data ready for the **Insight Creation Pipeline (ICP)**  

---

### **3. Insight Generation & Curation Layer (Pre-Analytics Engine)**

This layer executes computationally intensive analytics, transforms structured metrics into human-readable insights, and ensures every narrative is validated before reaching the RAG index.

#### Sub-components

| Sub-Component | Function | Example Tools |
|----------------|-----------|---------------|
| **Metric Engine** | Derives KPIs, deltas, aggregates | dbt metrics / Spark SQL |
| **Analytics Engine** | Detects trends, anomalies, correlations | Trino / Python / statsmodels |
| **Narrative Generation Engine** | Converts metrics → text summaries | Rule-based logic / small LLM |
| **Validation Engine** | Verifies factual & narrative consistency | Great Expectations + LLM validator |
| **Embedding Engine** | Generates embeddings and stores vectors | BERT / OpenAI / SentenceTransformers |
| **Registrar** | Writes verified insights & metadata into RAG store | FastAPI microservice |

#### Operational Cadence
| Pipeline Component | Refresh Frequency |
|--------------------|-------------------|
| Transactional ingestion | Hourly / Daily |
| Non-Transactional updates | Daily / Weekly |
| Pre-analytics batch | Nightly |
| RAG index update | Post-validation (automated) |

---

### **4. Insight Storage + RAG Index**

The persistent and searchable repository for verified insights.

- **Vector Database:** pgvector / Pinecone / Weaviate / Chroma  
- **Metadata Store:** Postgres / Elastic / Bigtable  
- **Schema Example:**
  ```json
  {
    "embedding": "...",
    "summary_text": "Royalty revenues grew 12% QoQ in LATAM.",
    "metadata": {
      "entity": "Sony Music",
      "period": "2025-Q3",
      "metric": "royalty_growth",
      "region": "LATAM",
      "source_data_id": "gold_table_X_row_Y",
      "validated_at": "2025-11-06T00:00:00Z"
    }
  }
  ```
- **Cost-Aware Embedding Strategy:** Use optimized open-source models (MiniLM, Instructor-BERT) for high-volume embedding to manage costs.

---

### **5. Insight API / Application Layer**

The real-time engine that powers insight retrieval, reasoning, and synthesis.

- **API:** REST / GraphQL (FastAPI, Node.js, Go)  
- **Retrieval Logic (Hybrid Search):**
  1. **FGAC Enforcement:** Apply user access controls via metadata filters (`metadata.entity IN (...)`).
  2. **Metadata Pre-filtering:** Use structured filters (e.g., `period='2025-Q3'`) to narrow the search.
  3. **Vector Search:** Semantic retrieval across the pre-filtered set.
  4. **Reasoning Engine (Small LLM):** Synthesizes top-k insights into a cohesive, grounded answer.
- **Output:** Synthesized answer + references (`source_data_id` citations).

---

### **6. User Interface / Dashboard / Bot**
- Web UI (Streamlit, React) or conversational interface (Slack bot)
- Controlled natural language or structured query templates for predictability
- Feedback capture: “Relevant” / “Not relevant” → logged for continuous improvement

---

### **7. Auto-Learning & Insight Coverage Expansion (Feedback Loop)**

- **Gap Logging:** Records unanswered questions, low-confidence, and negative feedback.  
- **Adaptive Learning:** Identifies missing metrics or dimensions in the RAG index.  
- **Recomputation Trigger:** Initiates targeted reprocessing in the Insight Creation Pipeline.  
- **Outcome:** Continuous RAG coverage improvement.

---

## 4. Governance & Observability

| Aspect | Practice | Enhancement |
|--------|-----------|-------------|
| **Data Catalog** | Register metrics/dimensions in dbt docs or OpenMetadata | Link Catalog → Insight Schema for unified definitions |
| **Lineage Tracking** | Airflow / Dagster lineage / DataHub | End-to-End traceability: User Query → RAG Chunk → GOLD Table ID |
| **Access Control** | Only aggregated data in RAG; FGAC in API layer | Metadata-enforced FGAC per request |
| **Validation** | Great Expectations + pre-embedding checks | Model-in-the-Loop Validation for factual correctness |
| **Cost Management** | Batch jobs, light retrieval | Cost-aware embeddings and model tiering |
| **Feedback Metrics** | Track unanswered queries and recall | Add: Coverage Expansion Rate + Answer Fidelity |
| **System Health Metrics** | Measure operational KPIs | See below table |

#### System Health KPIs

| Metric | Description | Target |
|---------|--------------|---------|
| Insight Freshness | Time since last successful batch | < 24 hours |
| Coverage Rate | % of questions answered from RAG | > 85% |
| Avg Latency | Retrieval + synthesis time | < 2 seconds |
| Cost per Answer | Embedding + LLM compute cost | Tracked weekly |

---

## 5. Deep Dive 1: Insight Creation Pipeline (ICP) Data Model & Transformation

### **A. GOLD Source Table Schema (Example)**

| Column Name | Data Type | Description | Purpose |
| :--- | :--- | :--- | :--- |
| `metric_id` | `VARCHAR` | Unique ID (e.g., `royalty_growth_q_q`) | Metric Engine output |
| `entity_id` | `VARCHAR` | Rightsholder or business unit ID | **FGAC/Metadata Tag** |
| `period` | `DATE` | End date of reporting period (e.g., `2025-09-30`) | **Metadata Tag** |
| `metric_value` | `NUMERIC(18,4)` | The calculated KPI value (e.g., `0.1200`) | Input for Narrative Engine |
| `metric_units` | `VARCHAR` | %, USD, count | Input for Narrative Engine |
| `comparison_period` | `VARCHAR` | QoQ, YoY, MoM | Input for Narrative Engine |
| `anomaly_score` | `NUMERIC(18,4)` | Score from Anomaly Detector | Adds emphasis for LLM or rule engine |
| `data_lineage` | `JSON` | Pointers to SILVER/BRONZE tables | **Traceability anchor** |

### **B. Transformation: Metric → Narrative → Chunk**

| Step | Output Example | Prompt/Logic | Notes |
| :--- | :--- | :--- | :--- |
| **1. Narrative Generation** | “Sony Music’s royalty revenue grew by 12% QoQ in LATAM.” | Template or LLM prompt referencing metric fields; prepend “🚨 Anomaly Detected” if anomaly_score > 0.8 | Small model or Jinja for consistency |
| **2. Validation (Factual)** | `PASS` / `FAIL` | Validate text vs metric_value/unit | Use Great Expectations + LLM validator |
| **3. Chunk Finalization** | JSON: summary_text + metadata | Combine validated summary + GOLD metadata | One GOLD row = one insight chunk |

---

## 6. Deep Dive 2: Fine-Grained Access Control (FGAC) Enforcement

### **A. Access Model: Relationship-Based Access Control (ReBAC)**

- Example: `user:john_doe` → `viewer` → `entity:sony_music`  
- Permission: Does the user have “viewer” access to entity X?

### **B. FGAC Enforcement Flow (Layer 5)**

| Step | API Action | RAG Index Interaction | Outcome |
|------|-------------|------------------------|----------|
| **1. Authentication** | User ID from token | — | User context established |
| **2. Authorization** | AuthZ service (Okta FGA / SpiceDB) returns allowed entities | — | `allowed_entities = ['Sony Music', 'UMG']` |
| **3. Hybrid Retrieval** | Construct query + apply metadata filter (`metadata.entity IN allowed_entities`) | Executes semantic + metadata search | Retrieves only authorized insights |
| **4. Synthesis** | Combine retrieved chunks via small LLM | — | Secure, grounded final response |

---

## 7. Example Tech Stack

| Layer | Tools |
|--------|--------|
| **Storage** | GCS / S3 + Iceberg / Delta |
| **Processing / Curation** | Spark / Trino / DuckDB / dbt |
| **Orchestration** | Airflow / Dagster |
| **Vector DB / Index** | pgvector / Pinecone / Weaviate / Chroma |
| **Feature / Metadata Catalog** | dbt metrics / Feast / DataHub |
| **LLM / Embeddings** | OpenAI / Anthropic (synthesis) + local model (embedding) |
| **API / RAG Core** | FastAPI / LangChain / BentoML |
| **UI** | Streamlit / React / Slack bot |

---

## 8. Benefits

✅ **Explainable and Traceable** — Grounded with `source_data_id` and lineage.  
✅ **Cost & Speed Optimized** — Heavy computation batched; retrieval lightweight.  
✅ **Adaptive and Resilient** — Feedback loop ensures continuous learning.  
✅ **Compliant & Secure** — FGAC ensures tenant-level isolation.  
✅ **Production-Ready** — Built-in observability, lineage, and cost tracking.

---

## 9. Evolution Path

| Stage | Goal | Example Tools |
|--------|------|----------------|
| **Stage 1** | Manual pre-analytics + static embeddings | DuckDB + pgvector |
| **Stage 2** | Automated metrics, anomaly jobs, hybrid search | Spark / Airflow + Pinecone / Postgres |
| **Stage 3** | Feedback-driven coverage expansion | Dagster + dbt + RAG retraining |
| **Stage 4** | Fully adaptive reasoning microservice | Feature store + advanced LLM orchestration |

---

## 10. Summary

The **AIG Framework v1.0** combines:  
- **Batch analytics** (ICP metrics & validation)  
- **Real-time RAG** (retrieval + synthesis)  
- **Continuous learning** (feedback & recomputation)  

Delivering **traceable, cost-efficient, and adaptive insights** for enterprise-scale AI systems.

---

*© 2025 Kirtan Bhatt. Adaptive Insight Generation (AIG) Architecture Framework Documentation
