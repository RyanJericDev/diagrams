# How Hybrid KB Search Works

The Knowledge Base (KB) uses **hybrid search** — combining **vector search** (semantic meaning) with **keyword scoring** (exact token matching) — to find the most relevant help articles for each customer ticket. Results from both methods are merged using **Reciprocal Rank Fusion (RRF)**, so articles that appear in both rankings are naturally boosted.

---

## The Big Picture

```mermaid
flowchart LR
    A["📄 KB Document\nUploaded by Admin"] --> B["🔍 Parsed into\nSections"]
    B --> C["🧩 Split into\nSmall Chunks"]
    C --> D["🧠 AI Converts to\nNumber Vectors"]
    D --> E[("🗄️ Vector\nDatabase")]

    F["🎫 Customer Ticket\nComes In"] --> G1["🧠 Vector Search\n(semantic meaning)"]
    F --> G2["🔤 Keyword Search\n(token matching)"]
    E --> G1
    G1 --> H{"🔀 Reciprocal Rank\nFusion (RRF)"}
    G2 --> H
    H --> I["📋 Top Matching\nKB Sections"]
    I --> J["🤖 AI Writes\nResponse Using\nMatched KB"]

    style A fill:#dbeafe,stroke:#3b82f6
    style F fill:#fef3c7,stroke:#f59e0b
    style E fill:#d1fae5,stroke:#10b981
    style H fill:#fce7f3,stroke:#ec4899
    style J fill:#ede9fe,stroke:#8b5cf6
```

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

    subgraph step5["Step 5 — Store"]
        F --> G["Vectors are saved in\nPostgreSQL with pgvector\nindexed for fast\nsimilarity search"]
    end

    style step1 fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
    style step2 fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style step3 fill:#fefce8,stroke:#eab308,stroke-width:1px
    style step4 fill:#fdf4ff,stroke:#a855f7,stroke-width:1px
    style step5 fill:#f0fdfa,stroke:#14b8a6,stroke-width:1px
```

---

## Step-by-Step: Finding Relevant KB Content for a Ticket

When a customer ticket is analyzed, the system finds the best KB content to inform the AI response using both search methods in parallel.

```mermaid
flowchart TD
    A["🎫 New ticket arrives\n(subject + body)"] --> B["Phase 1\nAI classifies the ticket\ninto a category\n(e.g. 'cancellation')"]

    B --> KW["🔤 Keyword Scoring\n(always runs, <1ms)"]
    B --> VS["🧠 Vector Search\n(Bedrock + pgvector)"]

    subgraph keyword["Keyword Scoring"]
        KW --> KW1["Tokenize ticket text\n(stopwords removed)"]
        KW1 --> KW2["Score each KB article by:\n• content token overlap\n• title token overlap\n• keyword/tag overlap\n• category match (+0.3)\n• universal boost (+0.15)"]
        KW2 --> KW3["Apply relative threshold\n(drop below 25% of top score)"]
    end

    subgraph vector["Vector Search"]
        VS --> VS1["Generate embedding\nvia AWS Bedrock Titan v2"]
        VS1 --> VS2{"Search pgvector\nfiltered by category\n+ source type"}
        VS2 -->|"3+ results"| VS3["De-duplicate\n(best chunk per source)"]
        VS2 -->|"< 3 results"| VS4["Retry without\ncategory filter"]
        VS4 --> VS3
    end

    KW3 --> RRF{"🔀 Reciprocal Rank Fusion\nscore = Σ 1/(60 + rank)\nacross both lists"}
    VS3 --> RRF

    VS2 -->|"Search fails"| KWONLY["Keyword-only\nfallback"]
    KWONLY --> RESULT

    RRF --> RESULT["📋 Top 10 merged results\n(items in both lists\nare naturally boosted)"]
    RESULT --> AI["🤖 AI drafts response\nusing matched KB"]

    RESULT -->|"Both empty"| FALLBACK["Ultimate fallback:\nfilter KB by\ncategory tags"]
    FALLBACK --> AI

    style A fill:#fef3c7,stroke:#f59e0b
    style B fill:#dbeafe,stroke:#3b82f6
    style RRF fill:#fce7f3,stroke:#ec4899
    style KWONLY fill:#fef3c7,stroke:#f59e0b
    style FALLBACK fill:#fee2e2,stroke:#ef4444
    style AI fill:#ede9fe,stroke:#8b5cf6
    style keyword fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style vector fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
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

RRF merges two ranked lists into one by assigning each item a score based on its position in each list. Items that appear in _both_ lists are naturally boosted without any weight tuning.

```mermaid
flowchart TD
    subgraph input["Two Ranked Lists for 'order cancellation refund'"]
        direction LR
        subgraph vec["Vector (semantic)"]
            V1["#1 Cancellation Policy (0.82)"]
            V2["#2 Refund Timelines (0.75)"]
            V3["#3 Return Process (0.71)"]
        end
        subgraph kw["Keyword (token match)"]
            K1["#1 Cancellation Policy (0.9)"]
            K2["#2 Order Status FAQ (0.7)"]
            K3["#3 Refund Timelines (0.6)"]
        end
    end

    subgraph rrf["RRF Scores  —  score = Σ 1/(60 + rank)"]
        R1["✅ Cancellation Policy\n1/61 + 1/61 = 0.0328\n(in BOTH lists → boosted)"]
        R2["✅ Refund Timelines\n1/62 + 1/63 = 0.0320\n(in BOTH lists → boosted)"]
        R3["Order Status FAQ\n1/62 = 0.0161\n(keyword only)"]
        R4["Return Process\n1/63 = 0.0159\n(vector only)"]
    end

    input --> rrf

    style input fill:#f8fafc,stroke:#94a3b8,stroke-width:1px
    style vec fill:#eff6ff,stroke:#3b82f6,stroke-width:1px
    style kw fill:#f0fdf4,stroke:#22c55e,stroke-width:1px
    style rrf fill:#fdf4ff,stroke:#a855f7,stroke-width:1px
    style R1 fill:#d1fae5,stroke:#10b981
    style R2 fill:#d1fae5,stroke:#10b981
```

## Key Details

| Setting | Value | What It Means |
|---|---|---|
| **Search method** | Hybrid (vector + keyword + RRF) | Both methods run, results fused |
| **Embedding model** | AWS Bedrock Titan v2 | The AI that converts text to vectors |
| **Vector size** | 1,024 numbers | How much meaning each vector captures |
| **Chunk size** | ~1,500 characters | How big each searchable piece of text is |
| **Chunk overlap** | 200 characters | Overlap between chunks to avoid losing context at boundaries |
| **Similarity threshold** | 0.3 (of 1.0) | Minimum relevance score for vector matches |
| **RRF constant (k)** | 60 | Standard value from the RRF paper; higher = less emphasis on top ranks |
| **Keyword threshold** | Relative (25% of top) | Adapts to score distribution instead of fixed cutoff |
| **Stopwords** | ~60 English words | Removed from tokenization to improve keyword signal |
| **Max results** | 10 | How many KB pieces are sent to the AI per ticket |
| **Index type** | HNSW | Fast approximate search that scales to large datasets |
| **Fallback chain** | Keyword-only → category filter | Graceful degradation if vector search unavailable |
