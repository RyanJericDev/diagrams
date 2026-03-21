# Draft Email Feedback System

## What Is This?

When a customer support ticket comes in, our AI writes a draft email response for the reviewer. This feature lets reviewers **rate those drafts** and leave feedback. The AI then **learns from that feedback** to write better drafts over time.

---

## How It Works

```mermaid
flowchart TD
    A["Customer submits a support ticket"] --> B["AI analyzes the ticket"]
    B --> C["AI checks past reviewer feedback"]
    C --> D["AI writes a draft email response"]
    D --> E["Reviewer sees the draft"]
    E --> F{"How is the draft?"}
    F -- "Looks good" --> G["Reviewer clicks 'Good'"]
    F -- "Could be better" --> H["Reviewer clicks 'Needs Improvement'\nand explains what to fix"]
    G --> I["Feedback is saved"]
    H --> I
    I -.-> |"AI reads this feedback\nwhen drafting future emails"| C

    style A fill:#e8f4fd,stroke:#4a90d9
    style D fill:#e8f4fd,stroke:#4a90d9
    style E fill:#fff3cd,stroke:#d4a017
    style G fill:#d4edda,stroke:#28a745
    style H fill:#f8d7da,stroke:#dc3545
    style I fill:#f0f0f0,stroke:#666
```

---

## The Learning Loop

```mermaid
flowchart LR
    A["AI writes draft"] --> B["Reviewer rates it"]
    B --> C["Feedback stored"]
    C --> D["Next time AI drafts\nfor similar tickets,\nit avoids past mistakes"]
    D --> A

    style A fill:#e8f4fd,stroke:#4a90d9
    style B fill:#fff3cd,stroke:#d4a017
    style C fill:#f0f0f0,stroke:#666
    style D fill:#d4edda,stroke:#28a745
```

> **Example:** If a reviewer flags that refund emails sound too robotic, future refund draft emails will use a warmer, more empathetic tone.

---

## What the Reviewer Sees

```mermaid
flowchart TD
    A["Draft email is displayed\nin the review panel"] --> B["Feedback section appears below"]
    B --> C{"Reviewer picks a rating"}
    C -- "Good" --> D["Click 'Submit Feedback'\nDone!"]
    C -- "Needs Improvement" --> E["A text box appears to\nexplain what should change"]
    E --> F["Click 'Submit Feedback'"]
    F --> G["Feedback saved\nReviewer can update later if needed"]
    D --> G

    style A fill:#e8f4fd,stroke:#4a90d9
    style E fill:#fff3cd,stroke:#d4a017
    style G fill:#d4edda,stroke:#28a745
```

---

## How the AI Uses Feedback

```mermaid
flowchart TD
    A["New ticket comes in\ne.g. a cancellation request"] --> B["System looks up recent feedback\nfor cancellation drafts"]
    B --> C{"Any 'Needs Improvement'\nfeedback found?"}
    C -- "Yes" --> D["AI is shown up to 5 examples of\npast drafts and what reviewers\nwanted changed"]
    C -- "No" --> E["AI writes the draft\nwithout extra guidance"]
    D --> F["AI writes a better draft,\navoiding flagged mistakes"]

    style A fill:#e8f4fd,stroke:#4a90d9
    style D fill:#fff3cd,stroke:#d4a017
    style F fill:#d4edda,stroke:#28a745
    style E fill:#f0f0f0,stroke:#666
```

---

## Key Points

- **One rating per ticket** — reviewers can update their feedback anytime
- **Category-aware** — feedback on cancellation drafts improves future cancellation drafts first
- **Automatic** — no manual setup needed; the AI picks up feedback as reviewers submit it
- **Privacy-safe** — only the draft excerpt and feedback text are shared with the AI, no customer data
