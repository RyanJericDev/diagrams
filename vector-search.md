# How Hybrid KB Search Works

The Knowledge Base (KB) uses **hybrid search** — combining **vector search** (semantic meaning) with **keyword scoring** (exact token matching) — to find the most relevant help articles for each customer ticket. Results from both methods are merged using **Reciprocal Rank Fusion (RRF)**, then optionally reranked by an LLM.

---

## The Big Picture

```mermaid
flowchart LR
    A["📄 KB Document\nUploaded by Admin"] --> B["🔍 Parsed into\nSections"]
    B --> C["🧩 Split into\nSmall Chunks"]
    C --> D["🧠 AI Converts to\nNumber Vectors"]
    D --> E[("🗄️ Vector\nDatabase")]

    F["🎫 Customer Ticket\nComes In"] --> P1["🏷️ Phase 1\nAI Classification"]
    P1 --> Q["📝 Reformulated\nSearch Queries"]
    Q --> G1["🧠 Vector Search\n(semantic meaning)"]
    Q --> G2["🔤 Keyword Search\n(token matching)"]
    E --> G1
    G1 --> H{"🔀 Reciprocal Rank\nFusion (RRF)"}
    G2 --> H
    H --> R["🎯 LLM Reranker\n(optional)"]
    R --> I["📋 Top Matching\nKB Sections"]
    I --> J["🤖 AI Writes\nResponse Using\nMatched KB"]
    J --> FB["📊 Retrieval\nFeedback Log"]

    style A fill:#dbeafe,stroke:#3b82f6
    style F fill:#fef3c7,stroke:#f59e0b
    style E fill:#d1fae5,stroke:#10b981
    style P1 fill:#dbeafe,stroke:#3b82f6
    style Q fill:#e0f2fe,stroke:#0ea5e9
    style H fill:#fce7f3,stroke:#ec4899
    style R fill:#fef3c7,stroke:#f59e0b
    style J fill:#ede9fe,stroke:#8b5cf6
    style FB fill:#f0fdf4,stroke:#22c55e
```

---

## Where Vector Search is Used

Vector search is **not** just used during ticket analysis. The system reads and writes to the vector database in three distinct flows:

```mermaid
flowchart TD
    subgraph write["✏️ Write Path — Embedding Storage"]
        W1["Admin creates/edits\nKB Article"] --> W2["embed_article()\nChunk → Embed → Store"]
        W3["Admin uploads\nDocument (.pdf/.docx/.txt)"] --> W4["Async Worker\nparse → sections → embed"]
        W5["Admin edits\nDocument Section"] --> W6["embed_document_sections()\nRe-chunk → Re-embed"]
    end

    subgraph read["🔍 Read Path — Search"]
        R1["Ticket Analysis\n(classify.py)"] --> R2["hybrid_kb_search()\nMulti-query vector+keyword"]
    end

    subgraph cleanup["🧹 Cleanup Path"]
        C1["Document replaced\nby new upload"] --> C2["Delete old section\nembeddings"]
        C3["Weekly scheduled\njob (EventBridge)"] --> C4["cleanup_orphaned_embeddings()"]
    end

    W2 --> DB[("🗄️ kb_embeddings\npgvector + HNSW")]
    W4 --> DB
    W6 --> DB
    R2 --> DB
    C2 --> DB
    C4 --> DB

    style write fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
    style read fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style cleanup fill:#fef3c7,stroke:#f59e0b,stroke-width:1px
    style DB fill:#d1fae5,stroke:#10b981
```

### Trigger Summary

| Trigger | Function | File | Direction |
|---------|----------|------|-----------|
| Create KB article | `embed_article()` | `routes/knowledge_base.py` | Write |
| Update KB article | `embed_article()` | `routes/knowledge_base.py` | Write |
| Upload document (small) | `embed_document_sections()` | `routes/settings.py` | Write |
| Upload document (large) | `embed_document_sections()` | `async_worker.py` | Write |
| Edit document section | `embed_document_sections()` | `routes/settings.py` | Write |
| Analyze ticket | `hybrid_kb_search()` → `search_kb_vectors()` | `pipeline/classify.py` | Read |
| Document replaced | `delete(KbEmbedding)` | `async_worker.py`, `settings.py` | Delete |
| Weekly cleanup | `_cleanup_orphaned_embeddings()` | `async_worker.py` | Delete |

---

## Step-by-Step: Storing Knowledge

When an admin uploads a document or creates a KB article, the system prepares it for smart search.

```mermaid
flowchart TD
    subgraph step1["Step 1 — Upload"]
        A["Admin uploads a\n.txt, .docx, or .pdf file\n(up to 30 MB)"] --> B["File stored\nin AWS S3"]
    end

    subgraph step2["Step 2 — Parse"]
        B --> C["System reads the file\nand splits it into sections\nusing headings, dividers,\nand formatting"]
        C --> D["Each section gets\nauto-detected categories\nand keywords"]
    end

    subgraph step3["Step 3 — Chunk"]
        D --> E["Long sections are split\ninto smaller overlapping\nchunks (~1,500 characters)\nso nothing important\nis cut off"]
    end

    subgraph step4["Step 4 — Embed"]
        E --> F["AWS Bedrock AI reads\neach chunk and converts\nit into a list of 1,024\nnumbers (a 'vector')\nthat captures its meaning"]
    end

    subgraph step5["Step 5 — Store with Metadata"]
        F --> G["Vectors saved in\nPostgreSQL with pgvector\nindexed for fast\nsimilarity search"]
        G --> G2["Chunk metadata stored:\narticleId, category,\nsourceUpdatedAt,\nsectionOrder, fileName"]
    end

    style step1 fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
    style step2 fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style step3 fill:#fefce8,stroke:#eab308,stroke-width:1px
    style step4 fill:#fdf4ff,stroke:#a855f7,stroke-width:1px
    style step5 fill:#f0fdfa,stroke:#14b8a6,stroke-width:1px
```

---

## Step-by-Step: Finding Relevant KB Content for a Ticket

When a customer ticket is analyzed, the system finds the best KB content through a multi-stage retrieval pipeline.

```mermaid
flowchart TD
    A["🎫 New ticket arrives\n(subject + body)"] --> B["Phase 1: AI classifies ticket\ninto a category + generates\n1-3 clean search queries"]

    B --> MQ["📝 Search Queries\n(reformulated by AI)\nStripped of emotion,\ntypos fixed, canonical terms"]

    MQ --> LOOP["For each query\n(up to 3):"]

    LOOP --> KW["🔤 Keyword Scoring\n(always runs, <1ms)"]
    LOOP --> VS["🧠 Vector Search\n(Bedrock + pgvector)"]

    subgraph keyword["Keyword Scoring"]
        KW --> KW1["Tokenize query text\n(stopwords removed)"]
        KW1 --> KW2["Score each KB article by:\n• content token overlap\n• title token overlap\n• keyword/tag overlap\n• category match (+0.3)\n• universal boost (+0.15)"]
        KW2 --> KW3["Apply relative threshold\n(drop below 25% of top score)"]
    end

    subgraph vector["Vector Search"]
        VS --> PUB["Filter: JOIN kb_articles\nwhere isPublished = true\n(unpublished excluded)"]
        PUB --> VS1["Generate embedding\nvia AWS Bedrock Titan v2"]
        VS1 --> VS2{"Search pgvector\nfiltered by category\n+ source type"}
        VS2 -->|"3+ results"| VS3["De-duplicate\n(best chunk per source)"]
        VS2 -->|"< 3 results"| VS4["Retry without\ncategory filter"]
        VS4 --> VS3
    end

    KW3 --> LISTS["Collect all ranked lists\n(2N lists for N queries)"]
    VS3 --> LISTS

    LISTS --> RRF{"🔀 Reciprocal Rank Fusion\nscore = Σ 1/(60 + rank)\nacross ALL lists"}

    VS2 -->|"Search fails"| KWONLY["Keyword-only\nfallback"]
    KWONLY --> RESULT

    RRF --> RERANK{"🎯 LLM Reranker\n(if ENABLE_LLM_RERANK=true\nand >3 candidates)"}
    RERANK -->|"Rerank succeeds"| RESULT
    RERANK -->|"Rerank fails/disabled"| RESULT

    RESULT["📋 Top 10 merged results\nwith metadata:\narticleId, category,\nsourceUpdatedAt"]

    RESULT --> AI["🤖 Phase 2: AI drafts response\nusing matched KB articles"]

    RESULT -->|"Both empty"| FALLBACK["Ultimate fallback:\nfilter KB by\ncategory tags"]
    FALLBACK --> AI

    AI --> FB["📊 Retrieval Feedback Log\nsearchQueries, articlesRetrieved,\nretrievalRank, outcome"]

    style A fill:#fef3c7,stroke:#f59e0b
    style B fill:#dbeafe,stroke:#3b82f6
    style MQ fill:#e0f2fe,stroke:#0ea5e9
    style RRF fill:#fce7f3,stroke:#ec4899
    style RERANK fill:#fef3c7,stroke:#f59e0b
    style KWONLY fill:#fef3c7,stroke:#f59e0b
    style FALLBACK fill:#fee2e2,stroke:#ef4444
    style AI fill:#ede9fe,stroke:#8b5cf6
    style FB fill:#d1fae5,stroke:#10b981
    style PUB fill:#d1fae5,stroke:#10b981
    style keyword fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style vector fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
```

---

## Query Reformulation

Raw ticket text (emotional language, typos, multi-issue threads) used to dilute search quality. Now the Phase 1 Claude call generates clean search queries alongside classification — zero additional latency cost.

```mermaid
flowchart LR
    subgraph before["❌ Before: Raw Ticket Text"]
        B1["'I am SO frustrated!! My fairy\nlight spirit tree doesnt work\nand the olive vine is also\nbroken. Order #578121.\nI want a replacement ASAP!!!'"]
    end

    subgraph after["✅ After: Reformulated Queries"]
        A1["Query 1: 'fairy light spirit tree\nnot working defective'"]
        A2["Query 2: 'fairy light olive vine\nnot lighting up'"]
        A3["Query 3: 'replacement refund\ndefective items order 578121'"]
    end

    before --> after

    style before fill:#fee2e2,stroke:#ef4444,stroke-width:1px
    style after fill:#d1fae5,stroke:#10b981,stroke-width:1px
```

Each query runs keyword + vector search independently. All ranked lists are fused through RRF, so multi-issue tickets find articles for **each** issue rather than a blended mess.

---

## LLM Reranking (Optional)

After RRF fusion, an optional LLM reranker can reorder candidates by actual relevance. Disabled by default for safe rollout.

```mermaid
flowchart LR
    RRF["RRF Fusion\n(rank-based scores)"] --> CHECK{"ENABLE_LLM_RERANK\n= true?"}
    CHECK -->|No| OUT["Use RRF order"]
    CHECK -->|Yes| LLM["Claude reads query +\ntop candidates\n(title + 500 char preview)"]
    LLM --> RANK["Returns article IDs\nin relevance order"]
    RANK --> OUT2["Reranked results"]
    LLM -->|"Fails"| OUT

    style CHECK fill:#fef3c7,stroke:#f59e0b
    style LLM fill:#ede9fe,stroke:#8b5cf6
```

| Detail | Value |
|--------|-------|
| **Trigger** | `ENABLE_LLM_RERANK=true` env var |
| **Min candidates** | >3 (skip if too few) |
| **Token budget** | ~1,650 tokens (~$0.005/call) |
| **Latency** | ~500-800ms |
| **Fallback** | Any failure → keep RRF order |

---

## Retrieval Feedback Loop

Every ticket analysis logs what was searched and retrieved. When agents approve or reject queue items, the outcome is recorded — enabling long-term retrieval quality analysis.

```mermaid
flowchart LR
    CLASSIFY["Ticket classified"] --> LOG["kb_retrieval_logs\ncreated with:\n• searchQueries\n• articlesRetrieved\n• retrievalRank\n• category"]
    LOG --> PENDING["outcome = 'pending'"]

    APPROVE["Agent approves\nqueue item"] --> UPDATE1["outcome → 'approved'"]
    REJECT["Agent rejects\nqueue item"] --> UPDATE2["outcome → 'rejected'"]
    AUTO["Auto-executed\n(Phase 1)"] --> UPDATE3["outcome → 'auto_executed'"]

    subgraph analysis["Future Analysis"]
        Q1["Which articles appear\nin rejected outcomes?"]
        Q2["Which search queries\nfind irrelevant content?"]
        Q3["Which categories have\npoor retrieval?"]
    end

    UPDATE2 --> analysis

    style LOG fill:#dbeafe,stroke:#3b82f6
    style analysis fill:#fef3c7,stroke:#f59e0b,stroke-width:1px
    style UPDATE2 fill:#fee2e2,stroke:#ef4444
```

---

## isPublished Safety Filter

Vector search now JOINs the source table to exclude unpublished articles. Previously, soft-deleted articles (`isPublished=false`) still had embeddings in `kb_embeddings` and could appear in search results.

```mermaid
flowchart LR
    Q["Vector query"] --> JOIN["JOIN kb_articles\nON source_id = article.id"]
    JOIN --> FILTER["WHERE is_published = true"]
    FILTER --> RESULTS["Only published\narticles returned"]

    UNPUB["Unpublished article\n(soft-deleted)"] -.->|"❌ Excluded"| FILTER

    style UNPUB fill:#fee2e2,stroke:#ef4444
    style FILTER fill:#d1fae5,stroke:#10b981
    style RESULTS fill:#d1fae5,stroke:#10b981
```

---

## What is a "Vector"?

A vector is a list of numbers that represents the _meaning_ of a piece of text. Texts with similar meanings have similar vectors — even if they use completely different words.

```mermaid
flowchart LR
    subgraph example["Example: Similarity Matching"]
        direction TB
        T1["'My order never showed up'"] --> V1["Vector: [0.82, 0.15, ...]"]
        T2["'Package delivery delays\nand missing shipments'"] --> V2["Vector: [0.80, 0.17, ...]"]
        T3["'How to change my\naccount password'"] --> V3["Vector: [0.12, 0.91, ...]"]

        V1 -. "✅ Very similar\n(high match)" .-> V2
        V1 -. "❌ Not similar\n(low match)" .-> V3
    end

    style T1 fill:#fef3c7,stroke:#f59e0b
    style T2 fill:#d1fae5,stroke:#10b981
    style T3 fill:#fee2e2,stroke:#ef4444
```

---

## How Reciprocal Rank Fusion (RRF) Works

RRF merges multiple ranked lists into one by assigning each item a score based on its position in each list. With multi-query search, there are 2N lists for N queries — items that appear across multiple queries and both search methods are heavily boosted.

```mermaid
flowchart TD
    subgraph input["Ranked Lists for 3 queries × 2 methods = up to 6 lists"]
        direction LR
        subgraph q1["Query 1: 'fairy light defective'"]
            V1["Vector: Policy A, Policy B"]
            K1["Keyword: Policy A, FAQ C"]
        end
        subgraph q2["Query 2: 'olive vine broken'"]
            V2["Vector: Policy B, Guide D"]
            K2["Keyword: Policy B, Guide D"]
        end
    end

    subgraph rrf["RRF Scores  —  score = Σ 1/(60 + rank) across ALL lists"]
        R1["✅ Policy B\nAppears in 3 lists → 0.049"]
        R2["✅ Policy A\nAppears in 2 lists → 0.033"]
        R3["Guide D\nAppears in 2 lists → 0.032"]
        R4["FAQ C\nAppears in 1 list → 0.016"]
    end

    input --> rrf

    style input fill:#f8fafc,stroke:#94a3b8,stroke-width:1px
    style q1 fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
    style q2 fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style rrf fill:#fdf4ff,stroke:#a855f7,stroke-width:1px
    style R1 fill:#d1fae5,stroke:#10b981
    style R2 fill:#d1fae5,stroke:#10b981
```

---

## Chunk Metadata

Each embedding now stores rich metadata for downstream use (AI citation, staleness detection).

| Source Type | Metadata Fields |
|-------------|----------------|
| **Article** | `articleId`, `title`, `category`, `sourceUpdatedAt` |
| **Document Section** | `sectionId`, `heading`, `sectionOrder`, `sourceFileName`, `sourceUpdatedAt` |

Existing embeddings get enriched metadata on next re-embed (any article edit triggers `embed_article`). No migration needed.

---

## Key Details

| Setting | Value | What It Means |
|---|---|---|
| **Search method** | Hybrid (vector + keyword + RRF + optional LLM rerank) | Multi-stage retrieval pipeline |
| **Query reformulation** | AI generates 1-3 clean queries from raw ticket | Better search precision, multi-issue coverage |
| **Embedding model** | AWS Bedrock Titan v2 | The AI that converts text to vectors |
| **Vector size** | 1,024 numbers | How much meaning each vector captures |
| **Chunk size** | ~1,500 characters | How big each searchable piece of text is |
| **Chunk overlap** | 200 characters | Overlap between chunks to avoid losing context at boundaries |
| **Similarity threshold** | 0.3 (of 1.0) | Minimum relevance score for vector matches |
| **isPublished filter** | JOIN kb_articles WHERE is_published = true | Unpublished articles excluded from results |
| **RRF constant (k)** | 60 | Standard value from the RRF paper; higher = less emphasis on top ranks |
| **Keyword threshold** | Relative (25% of top) | Adapts to score distribution instead of fixed cutoff |
| **Stopwords** | ~60 English words | Removed from tokenization to improve keyword signal |
| **Max results** | 10 | How many KB pieces are sent to the AI per ticket |
| **Index type** | HNSW | Fast approximate search that scales to large datasets |
| **LLM reranker** | Off by default (`ENABLE_LLM_RERANK=true` to enable) | Cross-encoder style relevance reranking |
| **Feedback tracking** | `kb_retrieval_logs` table | Tracks search queries, retrieved articles, and outcome |
| **Fallback chain** | Keyword-only → category filter | Graceful degradation if vector search unavailable |
