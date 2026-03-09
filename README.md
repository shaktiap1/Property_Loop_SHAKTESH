# Financial RAG System : End‑to‑End Architecture (In‑Depth)

This document explains the **complete architecture** of the financial question‑answering system.

The system is divided into **two major macro‑pipelines**:

1. **Offline / Build‑Time Pipeline**
   (Data → Embeddings → PCA → FAISS Indexes)
2. **Online / Runtime Pipeline**
   (Query → Intent → Retrieval → Generation)

Each macro‑pipeline is itself composed of **smaller, well‑defined sub‑architectures**.

---

## PIPELINE A  OFFLINE BUILD PIPELINE

> **Goal:** Convert raw financial data into optimized vector indexes ready for fast semantic search.

This pipeline is **deterministic, repeatable, and cost‑sensitive**.
It is executed **once** or on data refresh — **never per user query**.

---

## A1. Raw Data Ingestion Architecture

```text
CSV Files
├── trades.csv
└── holdings.csv
        ↓
Pandas DataFrames
```

**Key properties**:

* Data is treated as the **source of truth**
* No LLM involvement
* No transformation beyond loading

---

## A2. Schema Validation & Data Discipline Layer

```text
Raw DataFrame
      ↓
Schema Validator
      ↓
Validated DataFrame
```

**What happens here:**

* Mandatory column presence checks
* Strict date parsing (`TradeDate`, `AsOfDate`)
* Numeric enforcement (`Quantity`, `Price`, `Qty`, `MV_Base`, `PL_YTD`)
* Null rejection in critical fields

**Special handling (Trades):**

```text
Price ≤ 0
   ↓
Quarantine → trades_quarantine.csv
```

This ensures:

* Invalid financial records never pollute embeddings
* Auditability is preserved

---

## A3. Semantic Row Serialization Layer

```text
Validated Rows
      ↓
Row → Natural‑Language Description
      ↓
semantic_core (string)
```

Each row is converted into a **self‑contained semantic sentence**.

Example (Trades):

```
Trade event on 2024‑01‑15 for AlphaFund: BUY 100 units of AAPL at price 187.5 strategy Momentum
```

Example (Holdings):

```
Holding position as of 2024‑01‑31 for AlphaFund: Held 250 units of AAPL valued at 46875 with YTD P&L 4200
```

**Why this matters:**

* Vector embeddings operate on *meaning*, not tables
* Each row becomes independently searchable

---

## A4. Embedding Generation Architecture (Batch‑Optimized)

```text
semantic_core strings
      ↓
Batching (20 items)
      ↓
Gemini Embedding API
      ↓
float32 vectors
```

**Key optimizations:**

* Controlled batch size (free‑tier safe)
* Retry with backoff
* Dummy vector fallback to preserve row alignment

Vectors are persisted as:

```
vectors/float/
├── trades_embeddings.pkl
└── holdings_embeddings.pkl
```

This acts as a **safety checkpoint**.

---

## A5. Dimensionality Reduction (PCA via FAISS)

```text
768‑D Embeddings
      ↓
FAISS PCA Training
      ↓
256‑D Embeddings
```

**Important properties:**

* PCA is trained **once**
* Same transform applied to trades & holdings
* Stored using FAISS native format

Output:

```
vectors/pca_transform.faiss
```

---

## A6. Vector Indexing (IVF + PQ)

```text
PCA Vectors
      ↓
IVF Partitioning (nlist=64)
      ↓
Product Quantization (m=32, 8‑bit)
      ↓
FAISS Indexes
```

Two independent indexes are built:

```
vectors/
├── trades_index/index_pq.faiss
└── holdings_index/index_pq.faiss
```

**Why separate indexes:**

* Avoids cross‑domain noise
* Enables intent‑based routing
* Improves precision

---

## RESULT OF MACRO PIPELINE A

At this point, the system has:

* Clean validated data
* Semantic text per row
* Optimized vector embeddings
* Search‑ready FAISS indexes

The system is now **half‑ready**.

---

# PIPELINE B- ONLINE QUERY PIPELINE

> **Goal:** Answer user questions using only indexed financial data — safely and deterministically.

This pipeline runs **per user query**.

---

## B1. User Query Intake

```text
User Question
      ↓
Raw Query String
```

No preprocessing, no assumptions.

---

## B2. Semantic Caching Architecture (Redis)

```text
Query
  ↓
Hash(query)
  ↓
Redis Lookup
```

Cached artifacts:

* Query embeddings
* (Optionally) answers

**Outcome paths:**

```
Cache Hit  → Skip expensive calls
Cache Miss → Continue pipeline
```

---

## B3. Intent Detection & Routing Logic

```text
Query
  ↓
Heuristic Rules
  ↓ (fallback)
Gemini Flash Classifier
  ↓
Intent Plan
```

Possible intents:

* `TRADE_ONLY`
* `HOLDING_ONLY`
* `MIXED`
* `UNSUPPORTED`

The intent directly controls **which FAISS indexes are queried**.

---

## B4. Query Embedding & PCA Projection

```text
Query Text
      ↓
Embedding Model
      ↓
768‑D Vector
      ↓
PCA Transform
      ↓
256‑D Vector
```

This ensures **query vectors live in the same space** as indexed data.

---

## B5. FAISS Retrieval Architecture

```text
Query Vector
      ↓
FAISS Search (nprobe=8)
      ↓
Top‑K Row IDs
```

Separate searches occur for:

* Trades index
* Holdings index

Only indexes allowed by intent are queried.

---

## B6. Context Reconstruction Layer

```text
Row IDs
      ↓
CSV Row Lookup
      ↓
Textual Context Blocks
```

Context is reconstructed using **original validated CSV rows** — not embeddings.

This guarantees:

* Numerical accuracy
* No hallucinated values

---

## B7. Answer Generation (Guarded RAG)

```text
Question + Context
      ↓
Strict Prompt
      ↓
Gemini Flash
      ↓
Final Answer
```

**Hard rule:**
If context is empty or insufficient:

```
"Sorry can not find the answer"
```

No guessing. No external knowledge.

---

## RESULT OF MACRO PIPELINE B

The system produces:

* Context‑grounded answers
* Or explicit refusal when data is absent

---

# COMPLETE SYSTEM : COMPILED VIEW

```text
                ┌───────────────────────────┐
                │   OFFLINE PIPELINE (A)    │
                │                           │
Raw CSV ──► Validate ──► Serialize ──► Embed ──► PCA ──► FAISS
                │                           │
                └───────────────▲───────────┘
                                │
                                │ (Indexes + PCA)
                                │
                ┌───────────────┴───────────┐
                │   ONLINE PIPELINE (B)     │
                │                           │
User Query ──► Cache ──► Intent ──► Embed ──► Search ──► Context ──► Answer
                │                           │
                └───────────────────────────┘
```

---

## 🎯 Final Engineering Characteristics

* **Deterministic** (no agent chaos)
* **Auditable** (CSV → Answer traceable)
* **Cost‑efficient** (batching, caching, PQ)
* **Scalable** (separate offline/online paths)
* **Production‑safe** (no hallucination paths)

---

## 📌 Mental Model (One‑Line)

> **LLMs talk. FAISS searches. Pandas computes. Rules decide.**

This architecture reflects that principle end‑to‑end.

---

**Thankyou so much for giving your time to read this, Have a great time ahead :)**
