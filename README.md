# Taskify — Company Knowledge Assistant - (Personal project)

Taskify lets employees ask questions about their company's documents and receive
answers with citations to the original sources. Administrators manage company
accounts and knowledge; each account has its own saved, private conversation.

The application is implemented and runs locally with Docker Compose. Local setup
uses deterministic AI stubs without paid provider calls. Independent review, live
OpenAI quality and performance measurements, and human onboarding studies remain
required for final acceptance. See the [validation record](specs/001-company-knowledge-qa/validation.md).

## What Demo Here

> [!NOTE]
> [Watch at youtube](https://www.youtube.com/watch?v=-8gFCtiwjQA)


[![Demo Here](https://img.youtube.com/vi/-8gFCtiwjQA/maxresdefault.jpg)](https://www.youtube.com/watch?v=-8gFCtiwjQA)


## What it does

- Operators provision companies and their first administrators through a CLI.
- Administrators create user accounts and upload, rename, replace, or remove documents.
- Users browse ready documents, download authorized originals, and ask questions.
- Answers use current company knowledge and include authenticated source citations.
  Unsupported or ambiguous questions produce an explicit refusal, insufficient-information
  result, clarification, or processing failure.
- Conversations persist across reloads and logins. Accounts cannot read each other's
  chats, including administrators in the same company.

Supported originals are PDF (including scanned pages), DOCX, UTF-8 TXT, PNG, and
JPEG, from 1 byte through 20 MiB. English and Spanish OCR runs locally. Processing
has explicit resource limits, so an accepted upload can still fail processing.
A failed replacement preserves the previous ready document.

## Architecture

| Component | Stack | Owns |
| --- | --- | --- |
| [Identity](services/identity/README.md) | Python 3.14, FastAPI, PostgreSQL | Companies, accounts, passwords, sessions, operator provisioning |
| [Knowledge](services/knowledge/README.md) | Python 3.12, FastAPI, PostgreSQL/pgvector, local parsers/OCR | Original files, revisions, extraction, embeddings, ingestion jobs |
| [Chat](services/chat/README.md) | Python 3.12, FastAPI, PostgreSQL, LangGraph/LangChain | Private conversations, answer jobs, messages, citation references |
| [Web](frontend/README.md) | React, TypeScript, Vite, nginx, pnpm | Browser interface and same-origin API ingress |

Each backend owns a separate database with distinct runtime and migration roles.
Company and account access is enforced by application checks and PostgreSQL row-level
security. Services communicate through authenticated internal HTTP contracts rather
than reading another service's database. Only Knowledge mounts the private original-file
volume. Knowledge and Chat run separate workers for durable background jobs.

```text
.
├── frontend/          # Web application and browser tests
├── services/
│   ├── identity/      # Authentication and account management
│   ├── knowledge/     # Documents, ingestion, retrieval, and sources
│   └── chat/          # Saved conversations and answer generation
├── infra/             # Environment initialization, database bootstrap, recovery
├── tests/             # Composed contracts, fixtures, evaluation, and capacity
├── specs/             # Feature specification, design, contracts, and validation
└── compose.yaml       # Local application, workers, migrations, and test services
```

### Architecture map

These diagrams describe the implemented local Compose deployment and its code paths.
Amber identifies Identity, teal identifies Knowledge, and indigo identifies Chat;
rose marks rejected or failed work. Solid arrows show calls or processing steps;
dashed arrows show internal HTTP or supporting relationships, as labeled.

| View | What to follow |
| --- | --- |
| [Runtime topology](#runtime-topology) | Browser ingress, service boundaries, workers, databases, files, and providers |
| [Code and dependency direction](#code-and-dependency-direction) | Ports and adapters in the browser and each backend |
| [Identity and authorization](#identity-and-authorization) | Login, password rotation, CSRF, tenant context, and session expiry |
| [Document ingestion](#document-ingestion) | Accepted originals → isolated extraction → chunks → embeddings → publication |
| [Answer decision graph](#answer-decision-graph) | Scope → current evidence → structured claims → grounding verification |
| [Durable turns and source release](#durable-turns-and-source-release) | Idempotent acceptance, worker leases, final authorization, polling, and citations |
| [Service-owned data](#service-owned-data) | Principal entities, relationships, and persistence invariants |
| [Startup and recovery](#startup-and-recovery) | Migration gates, readiness, consistent backups, and abandoned work |

### Runtime topology

The three databases share one PostgreSQL 16 server, but each service has its own
credentials and schema ownership. Durable queues are tables in the owning database.
Internal calls use distinct API/worker bearer credentials; user authority is resolved
online through Identity. The browser reaches public routes through nginx on one origin.
This view shows deployment and storage; the sequence diagrams below expand internal calls.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryTextColor":"#0f172a","edgeLabelBackground":"#f8fafc","lineColor":"#64748b","clusterBkg":"#f8fafc","clusterBorder":"#cbd5e1"},"flowchart":{"curve":"basis","nodeSpacing":24,"rankSpacing":45}}}%%
flowchart TB
    browser["Employee / administrator browser<br/>React 19 + TypeScript SPA<br/>Chat · knowledge · admin screens"]

    edge["nginx · 127.0.0.1:8080<br/>Vite build + same-origin API proxy"]

        subgraph identity["IDENTITY · Python 3.14 / FastAPI"]
            ia["identity API<br/>Authentication · sessions · users"]
            idb[("identity database<br/>Companies · accounts · sessions")]
        end

        subgraph knowledge["KNOWLEDGE · Python 3.12 / FastAPI"]
            ka["knowledge API<br/>Documents · retrieval · sources"]
            kdb[("knowledge database + pgvector<br/>Revisions · blocks · vectors<br/>Durable ingestion_jobs")]
            kw["knowledge-worker<br/>taskify-ingest<br/>Ingestion + cleanup"]
            parser["Isolated Linux parser<br/>Native extraction + local OCR"]
            files[("Private originals<br/>./data/originals")]
        end

        subgraph chat["CHAT · Python 3.12 / FastAPI"]
            ca["chat API<br/>Private history · turns · citations"]
            cdb[("chat database<br/>Conversations · messages · citations<br/>Durable chat_turns")]
            cw["chat-worker<br/>taskify-answer<br/>LangGraph · up to 20 attempts at once"]
        end
    ai["AI_MODE=openai · optional provider calls<br/>OpenAI embeddings + Responses<br/>Backend-only keys"]

    browser --> edge
    edge -->|"auth / users"| ia
    edge -->|"knowledge"| ka
    edge -->|"chat"| ca
    ia --> idb
    ka --> kdb
    ca --> cdb
    kdb -->|"leased jobs"| kw
    cdb -->|"leased turns"| cw
    kw -->|"publish"| kdb
    cw -->|"commit"| cdb
    ka --- files
    kw --- files
    kw --> parser
    kw -->|"embed documents"| ai
    ka -->|"embed query"| ai
    cw -->|"scope / generate / verify"| ai

    classDef web fill:#eff6ff,stroke:#2563eb,color:#0f172a
    classDef identity fill:#fffbeb,stroke:#d97706,color:#78350f
    classDef knowledge fill:#f0fdfa,stroke:#0d9488,color:#134e4a
    classDef chat fill:#eef2ff,stroke:#6366f1,color:#312e81
    classDef external fill:#faf5ff,stroke:#9333ea,color:#581c87
    class browser,edge web
    class ia,idb identity
    class ka,kw,parser,kdb,files knowledge
    class ca,cw,cdb chat
    class ai external
```

`AI_MODE=stub` uses deterministic local answerers and embeddings without OpenAI calls.
Stub and OpenAI vectors have different pipeline-version suffixes. Tracing is separately
opt-in through LangSmith, with hidden inputs/outputs. PostgreSQL persists in
`./data/postgres`. nginx blocks `/internal`, `/health`, and `/metrics` and routes
`/api/v1/auth/*` and `/api/v1/users` to Identity, `/api/v1/knowledge` and its children
to Knowledge, and `/api/v1/chat` and its children to Chat.

All application data lives in the three databases plus the private originals;
Chat stores source references and never reads Knowledge's database or filesystem.

Implementation: [Compose](compose.yaml), [ingress](infra/nginx/default.conf), and the
[Identity](services/identity/src/taskify_identity/bootstrap.py),
[Knowledge](services/knowledge/src/taskify_knowledge/bootstrap.py), and
[Chat](services/chat/src/taskify_chat/bootstrap.py) composition roots.

### Code and dependency direction

Both browser and backend use ports and adapters. Arrows below represent **code
dependencies**, rather than network traffic. Application code defines the ports;
outbound adapters implement them. Composition roots supply concrete implementations.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryTextColor":"#0f172a","edgeLabelBackground":"#f8fafc","lineColor":"#64748b","clusterBkg":"#f8fafc","clusterBorder":"#cbd5e1"}}}%%
flowchart LR
    subgraph web["FRONTEND · frontend/src"]
        direction TB
        wc["composition/<br/>Dependencies · routes · session guards"]
        wi["adapters/inbound/react/<br/>Pages · components · subscriptions"]
        wa["application/<br/>AuthController · KnowledgeController<br/>ChatController · createCompanyUser"]
        wp["application/ports/<br/>Auth · users · knowledge · chat<br/>Clock · IDs · attachment viewer"]
        wd["domain/<br/>Owned values · validation · errors<br/>Identity / knowledge / chat types"]
        wo["adapters/outbound/<br/>Same-origin HTTP + strict DTO decoding<br/>Browser timers · IDs · downloads"]
        wc --> wi --> wa --> wd
        wa --> wp
        wo -.->|"implements"| wp
        wc -.->|"injects"| wo
    end

    subgraph backend["EACH BACKEND · services/{identity,knowledge,chat}/src"]
        direction TB
        bc["bootstrap.py<br/>Settings + dependency construction"]
        bi["adapters/inbound/<br/>FastAPI routes · DTO boundaries<br/>Operator CLI / workers where applicable"]
        ba["application/<br/>Use cases · authorization · transactions<br/>Ingestion / retrieval / turn orchestration"]
        bp["application/ports/<br/>Persistence · security · Identity HTTP<br/>Knowledge HTTP · extraction · AI"]
        bd["domain/<br/>Entities · value objects · invariants<br/>Revision / answer / session policies"]
        bo["adapters/outbound/<br/>SQLAlchemy / psycopg · filesystem / OCR<br/>HTTPX · LangChain / LangGraph"]
        bc --> bi --> ba --> bd
        ba --> bp
        bo -.->|"implements"| bp
        bc -.->|"injects"| bo
    end
    web ~~~ backend

    classDef composition fill:#f1f5f9,stroke:#64748b,color:#0f172a
    classDef adapter fill:#eff6ff,stroke:#2563eb,color:#1e3a8a
    classDef application fill:#eef2ff,stroke:#6366f1,color:#312e81
    classDef domain fill:#f0fdfa,stroke:#0d9488,color:#134e4a
    class wc,bc composition
    class wi,wo,bi,bo adapter
    class wa,wp,ba,bp application
    class wd,bd domain
```

Backend core imports only standard-library code and owned core modules; framework,
database, and provider libraries remain in adapters. Frontend core is independent of
React, Fetch, and browser storage. Import-linter contracts and
[frontend architecture rules](frontend/tooling/architecture.mjs) enforce these boundaries.
Account changes clear controllers and download URLs; generation counters and aborted
requests prevent late responses from repopulating another account's interface.

### Identity and authorization

Operators create companies and administrators through the CLI. Administrators can
create company users, whose role and company are assigned by the server. Every initial
password must be changed before protected Knowledge or Chat access is allowed.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryColor":"#fffbeb","primaryTextColor":"#78350f","primaryBorderColor":"#d97706","actorBkg":"#fffbeb","actorBorder":"#d97706","actorTextColor":"#78350f","signalColor":"#64748b","signalTextColor":"#0f172a","noteBkgColor":"#f1f5f9","noteTextColor":"#0f172a"},"sequence":{"mirrorActors":false,"wrap":true}}}%%
sequenceDiagram
    autonumber
    actor B as Browser
    participant I as Identity API
    participant ID as Identity DB
    participant P as Knowledge / Chat API
    participant PD as Owning service DB

    Note over B,P: Public HTTP passes through same-origin nginx
    B->>I: POST /api/v1/auth/login · email + password + exact Origin
    I->>ID: Lock HMAC-keyed login window and matching account
    I->>I: Verify Argon2id password and apply login pause policy
    I->>ID: Save random-token SHA-256 digest, credential version, CSRF secret
    I-->>B: HttpOnly session cookie + account + synchronizer CSRF token

    opt Initial password still active
        B->>I: PUT /api/v1/auth/password · Origin + CSRF
        I->>ID: Lock account and verify current password / session
        I->>ID: Atomic password/version change + revoke old sessions + issue replacement
        I-->>B: Replacement cookie + current CSRF token
    end

    B->>P: Protected mutation · cookie + Origin + X-CSRF-Token
    P->>P: Validate route input and exact Origin
    P->>I: POST /internal/v1/sessions/resolve · caller bearer + cookie token
    I->>ID: Check revocation, credential version, password state, idle expiry, CSRF
    I-->>P: Trusted session ID + account ID + company ID + role
    P->>P: Enforce user/admin operation policy
    P->>PD: Set trusted company context, and account context in Chat
    PD->>PD: Forced RLS + ownership / version / uniqueness constraints
    PD-->>P: Commit authorized operation
    P-->>B: Public result

    Note over B,ID: Explicit user activity may refresh idle time. Inactivity expires at 30 minutes
    Note over I,P: Polling / background resolution never refresh idle time. No positive auth cache
    Note over B,PD: Invalid sessions, CSRF, roles, or ownership fail before protected work is released
```

Five failed logins for a normalized email within 15 minutes impose a 15-minute pause,
including unknown emails. Workers resolve stored login-session UUIDs with background-only
credentials. Runtime database roles are nonowners without `BYPASSRLS`; narrow definer
functions support session lookup and queue claims before ordinary tenant transactions.

Implementation: [authentication](services/identity/src/taskify_identity/application/authenticate.py),
[password changes](services/identity/src/taskify_identity/application/change_password.py),
and [internal caller scopes](services/identity/src/taskify_identity/adapters/inbound/http/internal_sessions.py).

### Document ingestion

Acceptance makes work durable before returning `202`; publication happens later.
Original bytes are immutable. Expensive extraction and embedding happen outside long
database transactions, and the worker must still own its lease when it publishes.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryTextColor":"#0f172a","edgeLabelBackground":"#f8fafc","lineColor":"#64748b","clusterBkg":"#f8fafc","clusterBorder":"#cbd5e1"},"flowchart":{"curve":"basis","nodeSpacing":25,"rankSpacing":38}}}%%
flowchart TB
    subgraph accept["1 · ACCEPT · Knowledge API"]
        upload["Admin uploads or replaces an original<br/>Changed password + Origin + CSRF<br/>Replacement requires current If-Match ETag"]
        stage["Validate title, filename, media type, content<br/>1 byte–20 MiB · supported formats only<br/>Stage under private server-generated key"]
        integrity["Guard original; verify size + SHA-256<br/>Revalidate login / admin / CSRF after staging"]
        accepted["Atomic DB acceptance<br/>Item + processing revision + queued job<br/>Replacement retains previous active revision"]
        response["202 Accepted<br/>Admin polls revision status<br/>Ordinary users see ready documents only"]
        upload --> stage --> integrity --> accepted --> response
    end

    subgraph process["2 · PROCESS · knowledge-worker"]
        claim["Claim queued job or expired lease<br/>120 s lease · heartbeat every 8 s<br/>Increment lease_version and attempt count"]
        check["Check pipeline, candidate, removal, lease<br/>Verify original before extraction"]
        extract["Fresh Linux parser child<br/>512 MiB address space · 90 s CPU / wall<br/>32 MiB output · no inherited app secrets"]
        formats["PDF: pypdf; empty pages → PDFium + OCR<br/>DOCX: ordered paragraphs / tables / headings<br/>TXT: strict UTF-8 · images: Tesseract eng + spa"]
        normalize["Normalized extracted blocks<br/>NFC + normalized newlines + source-offset maps<br/>Page / line / section / table / image locators"]
        split["Recursive cl100k_base chunking<br/>At most 800 tokens · up to 100 overlap<br/>Chunks stay within their source block"]
        embed["Embed chunks in the accepted vector space<br/>1536 dimensions · text-embedding-3-small<br/>Batches ≤32 chunks / 25,600 tokens"]
        fence{"Still current candidate,<br/>not removed, and<br/>live lease fence?"}
        publish["Atomic publication under item lock<br/>Save blocks + chunks + vectors<br/>Revision ready · job succeeded<br/>Switch active pointer; supersede old revision"]
        discard["Discard stale attempt<br/>Cannot publish after removal or reclamation"]
        retry{"Transient failure and<br/>fewer than 4 claims?"}
        backoff["Requeue whole revision<br/>Exponential backoff + jitter"]
        failed["Revision / job failed<br/>Safe error code; no partial publication<br/>Previous ready revision remains available"]
        claim --> check --> extract --> formats --> normalize --> split --> embed
        embed -->|"recheck original size + SHA-256"| fence
        fence -->|"yes"| publish
        fence -->|"no"| discard
        extract -->|"parser error"| retry
        embed -->|"provider / processing error"| retry
        retry -->|"yes"| backoff --> claim
        retry -->|"no · or permanent error"| failed
    end

    accepted -->|"ingestion_jobs in Knowledge DB"| claim
    response ~~~ claim
    publish --> ready["Current ready original<br/>Company browsing · downloads · retrieval"]
    ready --> maintenance["Admin rename / replace / remove<br/>If-Match prevents stale edits<br/>Remove clears pointers and cancels pending work"]

    classDef knowledge fill:#f0fdfa,stroke:#0d9488,color:#134e4a
    classDef durable fill:#eef2ff,stroke:#6366f1,color:#312e81
    classDef decision fill:#fffbeb,stroke:#d97706,color:#78350f
    classDef failure fill:#fff1f2,stroke:#e11d48,color:#881337
    class upload,stage,integrity,check,extract,formats,normalize,split,embed,ready,maintenance knowledge
    class accepted,response,claim,publish,backoff durable
    class fence,retry decision
    class discard,failed failure
```

Decoded images/rendered pages are limited to 40 million pixels; DOCX archive entries
total at most 256 MiB uncompressed. Invalid, encrypted, unreadable, or oversized parser
inputs fail explicitly. A version/model-space mismatch also fails explicitly.

The worker runs conservative cleanup every minute: staging/orphans older than 24 hours
require file guards and a fresh reference check. Obsolete originals and derived data
require a locked visibility/job check before retirement. Revision metadata remains for
historical identity, and database outages preserve files for a later sweep.

Implementation: [acceptance](services/knowledge/src/taskify_knowledge/application/upload.py),
[processing](services/knowledge/src/taskify_knowledge/application/process_ingestion.py),
[parser](services/knowledge/src/taskify_knowledge/adapters/outbound/extraction_child.py),
[maintenance](services/knowledge/src/taskify_knowledge/application/maintain_knowledge.py),
and [cleanup](services/knowledge/src/taskify_knowledge/application/reconcile_storage.py).

### Answer decision graph

The Chat worker runs an acyclic LangGraph. Prior messages resolve the topic of a
follow-up; factual claims must come from freshly retrieved company evidence. Scope,
generation, and verification are separate structured model roles in OpenAI mode.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryTextColor":"#0f172a","edgeLabelBackground":"#f8fafc","lineColor":"#64748b","clusterBkg":"#f8fafc","clusterBorder":"#cbd5e1"},"flowchart":{"curve":"basis","nodeSpacing":22,"rankSpacing":38}}}%%
flowchart TB
    input["Current question · 1–4000 code points<br/>Newest complete prior messages · ≤6000 tokens<br/>Accepted mode / model / graph / prompt / profile"]
    scope["SCOPE role<br/>Resolve intent + standalone company query<br/>History supplies topic context only"]
    intent{"Scope decision"}
    refusal["refusal<br/>Unrelated request or prohibited action"]
    clarification["clarification<br/>Ambiguous / missing referent or query"]

    subgraph retrieval["FRESH KNOWLEDGE RETRIEVAL · authenticated worker HTTP"]
        resolve["Knowledge resolves accepting login online<br/>Trusted company derived from Identity"]
        query["Embed standalone query<br/>Active pipeline / model / 1536-dimension space"]
        search["Exact pgvector cosine search<br/>Filter company + not removed + active ready revision<br/>Reject incompatible vector spaces; return top 20"]
        evidence["Return pipeline + retrieval configuration versions<br/>Chat deduplicates to at most 8 chunks<br/>Nearest candidates still need relevance checks"]
        resolve --> query --> search --> evidence
    end

    found{"Any evidence?"}
    insufficient["insufficient_information<br/>No evidence or inadequate support"]
    generate["GENERATE role<br/>Structured claims + supporting chunk IDs<br/>May instead request clarification / report insufficiency"]
    membership{"Answer claims use only<br/>supplied evidence IDs?"}
    verify["VERIFY role<br/>Check support, qualifications, and conflicts<br/>Accept or reject; cannot rewrite the answer"]
    grounded{"Grounded?"}
    candidate["Verified answer candidate<br/>Cited claims + exact evidence references<br/>Application release checks still required"]
    failure["failure<br/>Invalid claims, failed grounding, dependency outage,<br/>incompatible configuration, or attempt timeout"]

    input --> scope --> intent
    intent -->|"unrelated / action_request"| refusal
    intent -->|"ambiguous / no query"| clarification
    intent -->|"company_information / mixed"| resolve
    evidence --> found
    found -->|"no"| insufficient
    found -->|"yes"| generate
    generate -->|"insufficient evidence"| insufficient
    generate -->|"needs a specific subject"| clarification
    generate -->|"proposed answer"| membership
    membership -->|"yes"| verify --> grounded
    membership -->|"no"| failure
    grounded -->|"yes"| candidate
    grounded -->|"no"| failure
    query -->|"embedding / configuration failure"| failure
    scope -->|"role error / timeout"| failure

    classDef chat fill:#eef2ff,stroke:#6366f1,color:#312e81
    classDef knowledge fill:#f0fdfa,stroke:#0d9488,color:#134e4a
    classDef decision fill:#fffbeb,stroke:#d97706,color:#78350f
    classDef failure fill:#fff1f2,stroke:#e11d48,color:#881337
    class input,scope,generate,verify,candidate chat
    class resolve,query,search,evidence knowledge
    class intent,found,membership,grounded,refusal,clarification,insufficient decision
    class failure failure
```

Mixed requests can answer the company-information part while explicitly declining
the prohibited part. Documents and history are untrusted input; they cannot authorize
actions or change ownership. All graph roles share a 60-second whole-attempt deadline,
a 12-step graph limit, and one transient provider retry. The graph has no tools,
back edges, checkpoints, stores, or result cache. SDK retries are disabled.

Implementation: [LangGraph workflow](services/chat/src/taskify_chat/adapters/outbound/langgraph_workflow.py),
[model roles](services/chat/src/taskify_chat/adapters/outbound/langchain_openai.py),
[prompts](services/chat/src/taskify_chat/adapters/outbound/prompts.py), and
[exact vector search](services/knowledge/src/taskify_knowledge/adapters/outbound/postgres/search.py).

### Durable turns and source release

One account owns one conversation across login sessions. A question and its queued
turn commit together; workers produce a candidate, while application code authorizes
the final saved response. The browser receives complete persisted results.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryColor":"#eef2ff","primaryTextColor":"#312e81","primaryBorderColor":"#6366f1","actorBkg":"#eef2ff","actorBorder":"#6366f1","actorTextColor":"#312e81","signalColor":"#64748b","signalTextColor":"#0f172a","noteBkgColor":"#f0fdfa","noteTextColor":"#134e4a"},"sequence":{"mirrorActors":false,"wrap":true}}}%%
sequenceDiagram
    autonumber
    actor B as Browser
    participant API as Chat API
    participant DB as Chat DB
    participant W as Chat worker
    participant I as Identity
    participant K as Knowledge
    participant G as LangGraph + answerer

    B->>API: POST message · client UUID + content + expected_last_sequence
    API->>I: Resolve current cookie and enforce Origin + CSRF
    API->>DB: Lock account/conversation and check identical replay first
    Note over API,DB: New sends require current sequence and no active turn. Same UUID with changed content conflicts
    API->>DB: Commit user message + queued turn + accepted AI configuration
    API-->>B: 202 · committed question + turn ID

    W->>DB: Claim queued / expired turn and increment fence / attempt
    Note over DB,W: 30 s lease · heartbeat every 4 s · at most 3 claims
    W->>DB: Read fresh own-account history and question under live fence
    W->>I: Revalidate accepting login UUID · background activity
    W->>G: Run fresh scope → retrieve → generate → verify attempt
    G->>K: Search using accepting login + standalone query
    K->>I: Resolve current login and company · background activity
    K-->>G: Current ready evidence + retrieval configuration version
    G-->>W: Answer candidate or safe non-answer outcome

    rect rgb(240, 253, 250)
        Note over DB,K: APPLICATION RELEASE CHECKPOINT · final Chat transaction holds the live turn fence
        W->>DB: Lock / check current lease owner, version, expiry
        W->>I: Revalidate accepting login and exact stored ownership
        opt Candidate contains a factual answer
            W->>W: Recheck claim membership and at most 8 exact citation triples
            W->>K: POST /internal/v1/sources/validate · purpose=release
            K->>I: Revalidate accepting login · background activity
            K->>K: One fresh SQL snapshot of item + revision + chunk availability
            K-->>W: Current labels + locators + original paths, or unavailable
        end
        alt Login, deadline, membership, or source check fails
            W->>W: Replace entire candidate with safe failure and discard citations
        else Candidate remains authorized
            W->>W: Construct saved answer / refusal / insufficiency / clarification
        end
        W->>DB: Recheck live fence and atomically save assistant message + citations + terminal turn
    end

    loop Every second while queued / running
        B->>API: GET /api/v1/chat/turns/{turn_id}
        API->>I: Validate session with activity=poll
        API->>DB: Read committed own-account state
        API-->>B: Pending status or complete saved result
    end

    B->>K: Open exact original · /api/v1/knowledge/{item_id}/source?revision_id=...
    K->>I: Reauthorize current browser session
    K->>K: Check exact active ready revision and verify original size / SHA-256
    K-->>B: Authorized original bytes, or source unavailable
```

The source snapshot defines release consistency: replacement/removal **before** that
snapshot blocks the factual answer. A later change permits the already validated
answer to remain saved, but affects future retrieval, history citation hydration,
and source opens. Old citations never point to a replacement file.

An expired or reclaimed worker cannot commit its candidate or reuse a prior source
receipt. Recovery reruns the entire graph with fresh history/evidence and the turn's
accepted answer settings; provider calls may repeat. Unique terminal-message constraints
prevent duplicate saved answers. Reloading the page restores history and resumes polling
without creating another turn. No draft tokens stream to the browser.

Implementation: [message acceptance](services/chat/src/taskify_chat/application/submit_message.py),
[release transaction](services/chat/src/taskify_chat/application/process_turn.py),
[source snapshot](services/knowledge/src/taskify_knowledge/application/validate_sources.py),
and [browser controller](frontend/src/application/chat.ts).

### Service-owned data

The diagrams below show principal persisted fields, rather than every column or
composite constraint. Relationships stay inside each database. Knowledge and Chat
carry Identity UUID references, and Chat citations carry Knowledge UUID references;
these are validated through service contracts, with no cross-database foreign keys.
All identifiers are UUIDs unless a different key type is shown; timestamps are UTC.

<details>
<summary><strong>Identity database · companies, credentials, and revocable sessions</strong></summary>

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","primaryColor":"#fffbeb","primaryTextColor":"#78350f","primaryBorderColor":"#d97706","lineColor":"#92400e","tertiaryColor":"#ffffff"}}}%%
erDiagram
    companies ||--o{ accounts : owns
    accounts ||--o{ login_sessions : authenticates

    companies {
        uuid id PK
        text name
    }
    accounts {
        uuid id PK
        uuid company_id FK
        text normalized_email UK "Globally unique"
        text role "admin or user"
        text password_hash "Argon2id"
        int credential_version
        boolean must_change_password
    }
    login_sessions {
        uuid id PK
        uuid account_id FK
        text token_digest UK "SHA-256 of opaque cookie token"
        text csrf_secret
        int credential_version
        timestamptz last_activity_at
        timestamptz revoked_at "Nullable"
    }
    login_attempt_windows {
        text email_key PK "HMAC of normalized email"
        timestamptz window_started_at
        int failure_count
        timestamptz blocked_until "Nullable"
    }
```

Login windows include unknown emails and intentionally do not reference accounts.
The account's current credential version invalidates old sessions after password change.

</details>

<details>
<summary><strong>Knowledge database · versioned originals, provenance, vectors, and ingestion</strong></summary>

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","primaryColor":"#f0fdfa","primaryTextColor":"#134e4a","primaryBorderColor":"#0d9488","lineColor":"#0f766e","tertiaryColor":"#ffffff"}}}%%
erDiagram
    knowledge_items ||--o{ knowledge_revisions : versions
    knowledge_revisions ||--o{ extracted_blocks : extracts
    extracted_blocks ||--o{ knowledge_chunks : splits
    knowledge_revisions ||--o{ ingestion_jobs : queues
    pipeline_configurations ||--o{ knowledge_revisions : configures
    pipeline_configurations ||--o{ knowledge_chunks : embeds
    pipeline_configurations ||--o{ ingestion_jobs : pins
    knowledge_revisions ||--o| retired_originals : retires

    knowledge_items {
        uuid id PK
        uuid company_id "Identity reference; RLS scope"
        text title
        int version "If-Match optimistic concurrency"
        uuid active_revision_id FK "Nullable; current ready revision"
        uuid candidate_revision_id FK "Nullable; processing or failed candidate"
        timestamptz removed_at "Nullable; visibility removed"
    }
    knowledge_revisions {
        uuid id PK
        uuid item_id FK
        uuid company_id
        text storage_key UK "Private immutable original"
        text sha256
        int size_bytes
        text state "processing, ready, failed, superseded, cancelled"
        text pipeline_version FK
    }
    extracted_blocks {
        uuid id PK
        uuid revision_id FK
        text normalized_text
        jsonb locator
        jsonb source_offset_map
        text extractor_version
    }
    knowledge_chunks {
        uuid id PK
        uuid item_id FK
        uuid revision_id FK
        uuid block_id FK
        text text
        int token_count "1-800"
        jsonb locator
        text pipeline_version FK
        text embedding_model
        vector embedding "1536 dimensions"
    }
    ingestion_jobs {
        uuid id PK
        uuid revision_id FK
        text pipeline_version FK
        text state "queued, running, succeeded, failed, cancelled"
        int attempt_count "At most 4"
        timestamptz run_after
        uuid lease_owner "Nullable"
        timestamptz lease_until "Nullable"
        int lease_version "Monotonic fence"
    }
    pipeline_configurations {
        text version PK "Immutable; includes embedding backend"
        text normalization_version
        jsonb extractor_versions
        text tokenizer "cl100k_base"
        int chunk_tokens "800"
        int overlap_tokens "100"
        text embedding_model "text-embedding-3-small"
        int dimensions "1536"
    }
    retired_originals {
        uuid revision_id PK,FK
        uuid company_id
        text storage_key UK
        timestamptz purged_at
    }
```

Tenant-owned rows carry `company_id`, including blocks, chunks, and jobs. Composite
foreign keys enforce exact item/revision/block ownership. Deferred guards require
active pointers to reference ready revisions; only one processing revision per item
and one job per revision/pipeline are allowed. Retiring bytes never relinks old sources.

</details>

<details>
<summary><strong>Chat database · private history, accepted turns, and exact citations</strong></summary>

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","primaryColor":"#eef2ff","primaryTextColor":"#312e81","primaryBorderColor":"#6366f1","lineColor":"#4f46e5","tertiaryColor":"#ffffff"}}}%%
erDiagram
    conversations ||--o{ chat_turns : accepts
    conversations ||--o{ chat_messages : orders
    chat_turns ||--|{ chat_messages : pairs
    chat_messages ||--o{ citations : supports

    conversations {
        uuid id PK
        uuid company_id "Identity reference"
        uuid account_id UK "One conversation per account; RLS scope"
        bigint last_sequence
    }
    chat_turns {
        uuid id PK
        uuid conversation_id FK
        uuid client_message_id "Unique within conversation"
        uuid accepted_login_session_id "Identity reference"
        uuid user_message_id FK,UK
        uuid assistant_message_id FK,UK "Nullable until terminal"
        text state "queued, running, completed, failed"
        int attempt_count "At most 3"
        bigint lease_version "Monotonic fence"
        uuid lease_owner "Nullable"
        timestamptz lease_until "Nullable"
        text ai_mode
        text model "Exact accepted model"
        text graph_version
        text prompt_version
        text model_config_version
        text retrieval_config_version "Null while pending"
    }
    chat_messages {
        uuid id PK
        uuid conversation_id FK
        uuid turn_id FK
        bigint sequence "Unique within conversation"
        text sender "user or assistant; unique per turn"
        text content "Immutable saved text"
        text outcome "answer, refusal, insufficient_information, clarification, failure"
    }
    citations {
        uuid id PK
        uuid assistant_message_id FK
        uuid item_id "Knowledge reference"
        uuid revision_id "Knowledge reference"
        uuid chunk_id "Knowledge reference"
        jsonb locator
        text title_at_answer
        timestamptz validated_at
    }
```

Every Chat row carries both `company_id` and `account_id`; forced RLS isolates accounts
even inside the same company. Each turn has one user message and at most one assistant
message, with exactly one assistant required at termination. A partial unique index
allows only one queued/running turn per conversation. Accepted AI settings and ownership
are immutable; citation labels and locators remain tied to their exact original revision.

</details>

Schema sources: [Identity migration](services/identity/migrations/versions/0001_identity.py),
[Knowledge migrations](services/knowledge/migrations/versions), and
[Chat migrations](services/chat/migrations/versions).

### Startup and recovery

Compose gates normal startup on database configuration, owned migrations, and API
readiness. Migration credentials belong to one-shot operator jobs; runtime services
receive their own runtime credentials. Readiness probes use local configuration and
first-party dependencies without invoking OpenAI or LangSmith.

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"sans-serif","fontSize":"14px","primaryTextColor":"#0f172a","edgeLabelBackground":"#f8fafc","lineColor":"#64748b","clusterBkg":"#f8fafc","clusterBorder":"#cbd5e1"},"flowchart":{"curve":"basis","nodeSpacing":25,"rankSpacing":38}}}%%
flowchart TB
    env["Private .env + built images<br/>Project-local database / originals directories"]
    pg["PostgreSQL TCP healthy<br/>First boot creates owned databases / roles<br/>Knowledge alone gets pgvector"]
    configure["postgres-configure · one-shot operator job<br/>Sync configured role passwords over private Unix socket<br/>Preserve existing data and grants"]
    im["identity-migrate<br/>alembic upgrade head"]
    km["knowledge-migrate<br/>alembic upgrade head"]
    cm["chat-migrate<br/>alembic upgrade head"]
    identity["Identity API ready<br/>Owned storage / schema / settings"]
    knowledge["Knowledge API + ingestion worker<br/>Wait for own migration + healthy Identity<br/>API checks storage / parser runtime / Identity"]
    chat["Chat API + answer worker<br/>Wait for own migration + healthy Identity / Knowledge<br/>API checks storage / model config / dependencies"]
    web["Web ingress starts<br/>All three APIs healthy<br/>SPA + committed public routes available"]

    subgraph recovery["CONSISTENT RECOVERY · operator-controlled maintenance window"]
        freeze["Stop ingress, APIs, workers<br/>Exclude CLI and other writers"]
        backup["One matching backup set<br/>Identity dump + Knowledge dump + Chat dump<br/>Private originals archive + version metadata"]
        restore["Restore together on isolated target<br/>Matching roles / server / images<br/>Preserve RLS, ACLs, and original identity"]
        validate["Verify schema, ownership, file hashes, pointers,<br/>cross-company/account denials, source freshness<br/>Start dependencies → readiness → ingress"]
        resume["Durable pending jobs survive<br/>Expired leases reclaimed with higher fence<br/>Whole attempts restart with fresh checks"]
        freeze --> backup --> restore --> validate --> resume
    end

    env --> pg --> configure
    configure --> im --> identity
    configure --> km --> knowledge
    configure --> cm --> chat
    identity --> knowledge --> chat
    identity --> web
    knowledge --> web
    chat --> web
    web -.->|"scheduled maintenance"| freeze

    classDef ops fill:#f1f5f9,stroke:#64748b,color:#0f172a
    classDef identity fill:#fffbeb,stroke:#d97706,color:#78350f
    classDef knowledge fill:#f0fdfa,stroke:#0d9488,color:#134e4a
    classDef chat fill:#eef2ff,stroke:#6366f1,color:#312e81
    classDef web fill:#eff6ff,stroke:#2563eb,color:#1e3a8a
    class env,pg,configure,freeze,backup,restore,validate,resume ops
    class im,identity identity
    class km,knowledge knowledge
    class cm,chat chat
    class web web
```

Backup consistency requires all three databases and their matching originals with
writes stopped throughout capture. Abandoned ingestion waits for its 120-second lease
boundary; abandoned turns wait for their 30-second boundary. Reclaimed attempts must
obtain fresh authorization/source checks and cannot reuse stale worker results.

Implementation and procedures: [database ownership](infra/postgres/README.md),
[local storage](infra/local-storage.md), [backup/restore](infra/backup-restore.md),
[tracing](infra/tracing.md), and the [restore exercise](tests/integration/test_restore_recovery.py).

## Run locally

For the containerized application, install Docker with Compose and Python 3 to
initialize configuration. Host development additionally uses uv, Python 3.14 for
Identity, Python 3.12 for Knowledge/Chat, Node.js 22.12+, and the pnpm version pinned
in [frontend/package.json](frontend/package.json).

If you previously ran this project with named data volumes, choose either a
[fresh start](infra/local-storage.md#start-empty-with-existing-named-volumes) or
[migration](infra/local-storage.md#move-existing-named-volume-data) before starting
services with the new mounts.

For a new installation, run these commands from this project directory:

```sh
python3 infra/init-env.py
docker compose build
mkdir -p data/postgres data/originals
docker compose run --rm --no-deps --user root --entrypoint chown knowledge 10001:10001 /data/originals
docker compose up -d
docker compose ps
```

The initializer creates a private, ignored `.env` with independent random development
secrets. It refuses to overwrite an existing file; skip that command if your environment
is already initialized. Subsequent starts need only `docker compose build` and
`docker compose up -d` (or combine them with `docker compose up -d --build`).

Compose waits for PostgreSQL, synchronizes the local database role passwords with
the configuration, and runs all three migrations before starting their APIs and
workers. The web app waits for API readiness. The `postgres-configure` and
`*-migrate` containers finish with `Exited (0)`; this is expected. Runtime APIs and
workers receive only their own runtime database credentials. Existing data directories
are preserved. Production password rotation requires coordinating database clients
and operator access separately.

Open **[http://localhost:8080](http://localhost:8080)**. PostgreSQL is available to host
development tools at `127.0.0.1:55432`. The web application and database bind to loopback;
production deployment requires HTTPS, secure cookies, and encrypted service traffic.

### Fix database password authentication failures

If a migration reports `password authentication failed`, an existing PostgreSQL
volume may still contain passwords from an earlier `.env`. PostgreSQL runs
`bootstrap.sql` only when creating a new database volume; changing `.env` or
rebuilding images does not update existing role passwords.

The local `postgres-configure` startup job handles this automatically using a
private Unix socket shared only with PostgreSQL. With the current Compose file,
build and start the application:

```sh
docker compose build
docker compose up -d
```

This preserves existing databases and documents. Keep the `data/` directory;
`docker compose down -v` does not delete these bind-mounted files. To inspect startup
failures, run
`docker compose logs postgres-configure identity-migrate knowledge-migrate chat-migrate`.

### Create the first administrator

```sh
docker compose run --rm identity taskify-admin create \
  --company-name "Your company" \
  --display-name "Your name" \
  --email admin@example.com
```

The CLI prompts for a hidden, confirmed initial password. Log in and change that
password before using the application. To create another administrator for an existing
company, replace `--company-name` with `--company-id UUID`.

Administrators create user accounts at `/admin/users` and manage documents at
`/admin/knowledge`. Users browse ready originals at `/knowledge` and ask questions
at `/chat`. New users must also change their initial password. Share initial credentials
through your company's trusted channel; there is no public signup.

### Everyday commands

```sh
docker compose ps
docker compose logs --tail=100 identity knowledge knowledge-worker chat chat-worker web
docker compose stop
```

After changing application code, run `docker compose build` and `docker compose up -d`.
PostgreSQL data and originals persist in this project's `data/postgres/` and
`data/originals/` directories through bind mounts. Both are excluded from Git and Docker
build contexts. See the [local storage guide](infra/local-storage.md) to move existing
named-volume data before starting with these mounts. The shared PostgreSQL socket is
temporary communication state and remains in a Docker-managed volume; images, build
cache, and container logs also remain managed by Docker. Follow the
[backup and restore guide](infra/backup-restore.md) for a consistent snapshot of all
three databases and original files.

## AI configuration

[.env.example](.env.example) documents the configuration variables. Keep provider keys
and service credentials in backend configuration, outside version control and browser bundles.

| Setting | Purpose |
| --- | --- |
| `AI_MODE=stub` | Default deterministic development mode; makes no provider calls |
| `AI_MODE=openai` | Enables real OpenAI embeddings and answers; requires explicit configuration |
| `OPENAI_API_KEY` | Backend-only provider key for OpenAI mode |
| `OPEN_AI_MODEL` | Required exact chat model in OpenAI mode; no default or fallback |
| `OPENAI_EMBEDDING_MODEL` | Independent Knowledge embedding model; configured as `text-embedding-3-small` |
| `EMBEDDING_DIMENSIONS` | Fixed at 1536 in the current pipeline |
| `PIPELINE_VERSION` | Version of the immutable ingestion and embedding pipeline |
| `PUBLIC_ORIGIN` | Exact browser origin used for mutation validation; locally `http://localhost:8080` |

Stub and OpenAI embeddings occupy distinct versioned vector spaces. Switching embedding
mode, model, or pipeline requires a new version and explicit reindexing or fresh fixtures.
Changing only the chat model does not require re-embedding. See the
[quickstart](specs/001-company-knowledge-qa/quickstart.md) before enabling paid processing.

LangSmith tracing is optional and disabled by default. Chat and Knowledge support
opt-in traces with hidden inputs/outputs; Identity has no tracing dependency.
See [LangSmith tracing setup and troubleshooting](infra/tracing.md) for the separate
LangSmith key, project, endpoint, workspace, expected spans, and verification steps.

If an upload ends with `processing_unavailable`, check
`docker compose logs --tail=100 knowledge-worker`. An
`embedding_unavailable error_type=AuthenticationError status=401` means OpenAI rejected
the worker's key. Correct `OPENAI_API_KEY` in the private `.env`, then run
`docker compose up -d --wait` to recreate services with the updated environment;
`docker compose restart` does not reload `.env`. Upload again after correcting the key;
failed jobs have exhausted their automatic retries. For PostgreSQL's
`remaining connection slots are reserved` error, see the
[connection budget](infra/postgres/README.md).

## Development and checks

Install locked backend dependencies with each service's Python version:

```sh
uv sync --project services/identity --python 3.14 --locked
uv sync --project services/knowledge --python 3.12 --locked
uv sync --project services/chat --python 3.12 --locked
```

With local PostgreSQL initialized, run the owned backend suites:

```sh
(cd services/identity && TASKIFY_RUN_DB_TESTS=1 .venv/bin/pytest tests -q)
docker compose run --rm --build knowledge-test
(cd services/chat && TASKIFY_RUN_DB_TESTS=1 .venv/bin/pytest tests -q)
```

Knowledge's strict parser isolation requires Linux. Use the Docker test service;
macOS parser skips do not establish parser acceptance. Database fixtures target only
the dedicated service test databases.

Run static checks from each service directory so its configuration is applied:

```sh
uv run ruff check .
uv run ruff format --check .
uv run mypy src
uv run lint-imports
```

Use pnpm for frontend dependencies and checks:

```sh
pnpm --dir frontend install --frozen-lockfile
pnpm --dir frontend typecheck
pnpm --dir frontend lint
pnpm --dir frontend test --run
pnpm --dir frontend build
```

The [frontend guide](frontend/README.md) covers the development server and browser-test
credentials. The [quickstart](specs/001-company-knowledge-qa/quickstart.md) covers composed
contract tests, fixture provisioning, and isolated restore checks.

Recorded engineering validation includes 136 backend tests, 26 frontend tests, ten
real-stack browser checks, composed API contracts, and backup restoration. These results
use deterministic provider substitutes where applicable. Live-provider evaluation,
independent grading, capacity measurements, and onboarding with at least ten first-time
administrators and ten users remain open; see the
[validation record](specs/001-company-knowledge-qa/validation.md) for evidence and limitations.

## Documentation

| Guide | Contents |
| --- | --- |
| [Quickstart](specs/001-company-knowledge-qa/quickstart.md) | Full setup, checks, fixtures, and acceptance workflow |
| [Feature specification](specs/001-company-knowledge-qa/spec.md) | Stories, requirements, and success criteria |
| [Implementation plan](specs/001-company-knowledge-qa/plan.md) | Design, boundaries, and technical decisions |
| [Task list](specs/001-company-knowledge-qa/tasks.md) | Implementation progress and remaining acceptance work |
| [Data model](specs/001-company-knowledge-qa/data-model.md) | Service-owned entities and invariants |
| [Public HTTP contract](specs/001-company-knowledge-qa/contracts/http.md) / [OpenAPI](specs/001-company-knowledge-qa/contracts/openapi.json) | Public operations, schemas, and errors |
| [Internal contracts](specs/001-company-knowledge-qa/contracts/internal.md) | Authenticated service-to-service interfaces |
| [Frontend contract](specs/001-company-knowledge-qa/contracts/frontend.md) | Browser flows and behavior |
| [Operator CLI](specs/001-company-knowledge-qa/contracts/cli.md) | Company and administrator provisioning |
| [Backup and restore](infra/backup-restore.md) | Consistent database and original-file recovery |
| [Quality evaluation](tests/evaluation/README.md) | Reference review and opt-in live-provider grading |
| [Capacity and onboarding](tests/load/README.md) | Load measurements and human-study requirements |
| [Validation record](specs/001-company-knowledge-qa/validation.md) | Executed checks, measured results, and outstanding gates |
| [Project constitution](.specify/memory/constitution.md) | Engineering principles and acceptance requirements |
