# Draft Email Feedback System

## Overview

Reviewers rate AI-generated draft emails and provide feedback. Recent negative feedback is injected into the AI prompt so future drafts improve over time.

## End-to-End Flow

```mermaid
flowchart TD
    A[Ticket enters classification pipeline] --> B[Phase 1: Classify ticket category]
    B --> C[Load recent draft email feedback]
    C --> D{Feedback exists?}
    D -- Yes --> E[Append feedback examples to Phase 2 system prompt]
    D -- No --> F[Phase 2: Claude generates analysis + draftEmail]
    E --> F
    F --> G[Queue item created with draftEmail]
    G --> H[Reviewer sees draft in Queue Detail Panel]
    H --> I{Reviewer rates draft}
    I -- Good --> J[POST /draft-feedback rating=good]
    I -- Needs Improvement --> K[Reviewer writes feedback text]
    K --> L[POST /draft-feedback rating=needs_improvement]
    J --> M[(draft_email_feedback table)]
    L --> M
    M -.->|Next ticket in same category| C
```

## Feedback Loading Strategy

```mermaid
flowchart TD
    A[_load_recent_draft_feedback called] --> B[Query needs_improvement feedback<br/>with non-empty text]
    B --> C{Same category<br/>feedback available?}
    C -- Yes --> D[Load up to 5 same-category entries]
    C -- No --> E[Fall back to any category]
    D --> F{Less than 5?}
    F -- Yes --> G[Fill remaining from any category]
    F -- No --> H[Return feedback list]
    E --> H
    G --> H
    H --> I[Truncate drafts + feedback to 500 chars each]
    I --> J[Return as context for AI prompt]
```

## AI Prompt Injection

```mermaid
sequenceDiagram
    participant P as classify.py
    participant DB as Database
    participant AI as Claude API

    P->>DB: Query recent needs_improvement feedback
    DB-->>P: Feedback list (max 5)
    P->>P: Build Phase 2 system prompt
    P->>P: Append feedback examples to prompt
    Note over P: "IMPORTANT: Reviewers have flagged<br/>issues with previous draft emails..."
    P->>AI: System prompt + ticket context
    AI-->>P: Analysis + improved draftEmail
```

## API Endpoints

```mermaid
sequenceDiagram
    participant UI as Frontend
    participant API as Flask API
    participant DB as Database

    Note over UI,DB: On mount — load existing feedback
    UI->>API: GET /api/request-queue/{id}/draft-feedback
    API->>DB: Query by queue_item_id
    DB-->>API: Feedback row or null
    API-->>UI: { data: feedback | null }

    Note over UI,DB: On submit — upsert feedback
    UI->>API: POST /api/request-queue/{id}/draft-feedback
    Note over UI: { rating, feedbackText }
    API->>DB: Check existing feedback for queue item
    alt Existing feedback
        API->>DB: Update rating + text
    else New feedback
        API->>DB: Insert new row
    end
    DB-->>API: Saved row
    API-->>UI: { data: feedback }
```

## Data Model

```mermaid
erDiagram
    request_queue ||--o| draft_email_feedback : "has feedback"

    draft_email_feedback {
        int id PK
        int queue_item_id FK
        varchar rating "good | needs_improvement"
        text feedback_text "nullable"
        varchar category "from queue item"
        varchar submitted_by "reviewer email"
        timestamp created_at
    }

    request_queue {
        int id PK
        text draftEmail "AI-generated draft"
        varchar category
        varchar status
    }
```

## Frontend Component

```mermaid
stateDiagram-v2
    [*] --> Loading: Component mounts
    Loading --> NoFeedback: GET returns null
    Loading --> Saved: GET returns existing feedback

    NoFeedback --> GoodSelected: Click "Good"
    NoFeedback --> NeedsImprovementSelected: Click "Needs Improvement"

    Saved --> GoodSelected: Click "Good" (re-edit)
    Saved --> NeedsImprovementSelected: Click "Needs Improvement" (re-edit)

    GoodSelected --> Submitting: Click "Submit Feedback"
    NeedsImprovementSelected --> TextEntry: Textarea appears
    TextEntry --> Submitting: Click "Submit Feedback"

    Submitting --> Saved: POST succeeds
