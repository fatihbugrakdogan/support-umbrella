# Architecture

Companion to the [root README](../README.md); diagrams that don't fit the narrative there.
Excalidraw sources: [`diagrams/plan.excalidraw`](diagrams/plan.excalidraw) (the 8-day
gated plan), [`diagrams/decisions.excalidraw`](diagrams/decisions.excalidraw) (every major
option considered, rejected and chosen, with reasons).

## Component boundaries

```mermaid
flowchart TB
    subgraph dashboard["dashboard/ (Next.js)"]
        UI[list · detail · review panel]
        APIC["lib/api.ts — typed DTO client"]
    end
    subgraph backend["backend/ (FastAPI)"]
        RT["routes.py (thin)"]
        SVC["services.TicketService"]
        REPO["TicketRepository (protocol)"]
        MONGO[("MongoDB<br/>tickets · analyses · drafts · events")]
        MEM["InMemoryTicketRepository (tests)"]
    end
    subgraph pipeline["pipeline/ (no web deps)"]
        CFG["config_loader — frozen pipeline.yaml"]
        STR["ExtractionStrategy (protocol)<br/>single · cascade · council"]
        EXT[TicketExtractor]
        DRA[ReplyDrafter]
        ROU[ConfidenceRouter]
        LLM["LLMClient (protocol)"]
        ORC[OpenRouterClient]
        FAKE["FakeLLMClient (tests)"]
    end
    subgraph evalh["eval/ (offline)"]
        RUN["run.py — split discipline"]
        MET[metrics · charts]
        RUNS[("runs/ — committed artifacts")]
    end

    UI --> APIC --> RT --> SVC
    SVC --> REPO
    REPO -.implemented by.-> MONGO
    REPO -.implemented by.-> MEM
    SVC --> STR --> EXT
    SVC --> DRA & ROU
    CFG --> SVC
    EXT & DRA --> LLM
    LLM -.implemented by.-> ORC
    LLM -.implemented by.-> FAKE
    RUN --> EXT
    RUN --> MET --> RUNS
    RUNS -- "decision memo" --> CFG
```

Three protocols carry the whole design: swap `LLMClient` and the suite runs without a
network; swap `TicketRepository` and the database changes without touching business logic
(which is how Postgres → MongoDB happened mid-project in one commit); swap
`ExtractionStrategy` and a two-model cascade replaces a single model without the service
layer noticing (which is exactly how config v2/v3 shipped).

## Data model (documents)

```mermaid
erDiagram
    TICKET ||--o{ ANALYSIS : "append-only"
    TICKET ||--o{ DRAFT : "append-only"
    TICKET ||--o{ EVENT : "one per step"
    ANALYSIS ||--o{ DRAFT : "drafted from"

    TICKET {
        string id PK
        string subject
        string body
        string status "received -> processed"
        string review_state "pending -> approved"
    }
    ANALYSIS {
        string id PK
        string language
        string category
        string priority
        float category_confidence
        float priority_confidence
        float min_confidence "routing signal"
        bool needs_review
        float threshold "frozen 0.65"
        string served_model "recorded from response"
        string prompt_version
    }
    DRAFT {
        string id PK
        string body "customer's language"
        string served_model
        int latency_ms
    }
    EVENT {
        string type "received|extraction_*|routed|draft_*|reviewed"
        json payload
    }
```

The audit trail is **structural**: `analyses`, `drafts` and `events` have no update path
anywhere in the code — reprocessing appends. A reviewer's corrections live in the
`reviewed` event's payload, so the AI's original answer and the human's override are both
permanent.

## Eval methodology

```mermaid
flowchart TD
    SPEC["specs: labels fixed FIRST<br/>(category · priority+cue · language · difficulty · scenario)"]
    GEN["non-candidate generator writes tickets<br/>(grok-4.5 ≠ any candidate family)"]
    SPLIT["splits assigned at spec time<br/>dev 150 · selection 300 · holdout 150"]
    DEV["dev: prompt iteration v2→v4<br/>error analysis per version"]
    FREEZE["freeze prompt in changelog"]
    SEL["selection: 26-model matrix<br/>+ 1,106-strategy simulation"]
    COST["routed-cost objective<br/>API + review·$0.50 + graded error costs"]
    MEMO["decision memo + frozen config"]
    HOLD["holdout: runs exactly once<br/>for final reported numbers"]

    SPEC --> GEN --> SPLIT
    SPLIT --> DEV --> FREEZE --> SEL --> COST --> MEMO -.->|"later, once"| HOLD
```

Guardrails that exist as *code*, not policy: selection refuses non-frozen prompts;
holdout refuses to run twice or without the memo (both in eval/run.py and
scripts/run_holdout.py); ground-truth fields can't reach prompts
(they're never in the render path); an all-failures run aborts instead of emitting
plausible-but-empty metrics; the harness checks the OpenRouter balance before launching.
