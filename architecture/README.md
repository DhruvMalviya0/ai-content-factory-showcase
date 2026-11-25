# Architecture

Sanitised diagrams of the architecture **implemented today** in the private repository (version 0.3.0, inspected 2026-11-25).
They show components and flow, not code. Where a connection is not built, the diagram does not draw it.

| Diagram | Section |
|---|---|
| Overall system architecture | [1](#1-overall-system-architecture) |
| Media acquisition workflow | [2](#2-media-acquisition-workflow) |
| Media processing workflow | [3](#3-media-processing-workflow) |
| Queue and job lifecycle | [4](#4-queue-and-job-lifecycle) |
| AI provider fallback | [5](#5-ai-provider-fallback) |
| Autonomous orchestration | [6](#6-autonomous-orchestration) |

Written explanations: [SYSTEM_ARCHITECTURE](../documentation/SYSTEM_ARCHITECTURE.md),
[MEDIA_ACQUISITION](../documentation/MEDIA_ACQUISITION.md), [PROCESSING_PIPELINE](../documentation/PROCESSING_PIPELINE.md),
[QUEUE_AND_ORCHESTRATION](../documentation/QUEUE_AND_ORCHESTRATION.md),
[AI_PROVIDER_ARCHITECTURE](../documentation/AI_PROVIDER_ARCHITECTURE.md).

## 1. Overall system architecture

One computer, two processes, one database. The browser talks only to the dashboard, which proxies to the backend.

```mermaid
flowchart TB
    B[Browser] --> D[Dashboard<br/>Next.js, localhost only]
    D -->|same-origin API proxy| API[Backend API<br/>FastAPI, localhost only]
    API --> DB[(SQLite, WAL<br/>42 tables, 12 migrations)]
    API --> AQ[AI task queue]
    API --> MQ[Media job queue]
    subgraph Engines
        direction TB
        RS[Research and claim verification]
        SC[Script generation and checks]
        ML[Media library and import]
        TR[Transcription]
        MD[Moment discovery]
        CR[Clip rendering]
        AC[Source acquisition]
        AU[Autonomy scheduler]
        CA[Campaign records]
    end
    API --> Engines
    AQ --> TW[AI task worker]
    TW --> PM[Provider manager<br/>fallback, quotas, circuit breakers]
    PM --> EXT[Gemini and Groq<br/>official SDKs, optional]
    MQ --> MW[Media worker<br/>one job at a time]
    MW --> FF[FFmpeg and ffprobe<br/>supervised child processes]
    MW --> PM
    AC -.->|metadata only| YT[YouTube Data API<br/>optional]
```

Not drawn because not built: a video-assembly engine, voiceover, captions, export, publishing, and providers for any platform
other than YouTube (metadata only).

## 2. Media acquisition workflow

```mermaid
flowchart TD
    U[User action or<br/>autonomous discovery] --> R{Rights statement<br/>present?}
    R -- no --> STOP[Stops: authorisation required]
    R -- yes --> M[Adapter or provider decides the method]
    M -- file in import folder --> LI[Import in place<br/>read only]
    M -- plain HTTP link or feed enclosure --> DL[Download job<br/>guarded, resumable]
    M -- provider can fetch --> PF[Provider fetch job<br/>contract only; test double]
    M -- YouTube link --> YM[Validate link, fetch metadata<br/>wait for a supplied file]
    M -- no permitted method --> NS[Marked unavailable]
    YM -- user supplies file --> LI
    LI --> IMP[footage.import job<br/>copy, SHA-256, ffprobe, thumbnail]
    DL --> IMP
    PF --> IMP
    IMP --> LIB[(Media library<br/>content-addressed)]
```

Status: manual import and folder-source discovery verified live; links, feeds and channel discovery tested with mocks only;
YouTube download does not exist by design; other platforms not built.

## 3. Media processing workflow

```mermaid
flowchart LR
    LIB[(Library item<br/>ready)] --> T{Consent for this footage<br/>and a configured provider?}
    T -- no --> WAIT[Waits, with the reason shown]
    T -- yes --> TR[transcribe.media<br/>10-minute FLAC chunks]
    TR --> TS[(Transcript and segments)]
    TS --> MD[discover.moments<br/>text windows, local validation]
    MD --> MC[(Moment candidates<br/>scored, with reasons)]
    MC --> QF[Quality filter<br/>local, autonomous mode]
    MC --> USER[User picks a moment]
    QF --> RC[render.clip<br/>FFmpeg re-encode]
    USER --> RC
    RC --> V[ffprobe validation against the plan]
    V --> CL[(Clip file and record)]
    CL --> RV[Human review<br/>approve, reject, needs review]
```

## 4. Queue and job lifecycle

```mermaid
stateDiagram-v2
    [*] --> queued
    queued --> running: claim
    running --> succeeded: done
    running --> waiting: low disk, no attempt spent
    waiting --> queued: resume after
    running --> retry_wait: handler error, attempt spent
    retry_wait --> queued: resume after
    running --> failed: fatal, or attempts exhausted
    running --> paused: pause requested, at a safe point
    paused --> queued: resume
    running --> cancelling: cancel requested
    cancelling --> cancelled
    queued --> cancelled
    running --> queued: restart or shutdown recovery
    queued --> failed: crash-loop guard
    succeeded --> [*]
    failed --> [*]
    cancelled --> [*]
```

## 5. AI provider fallback

```mermaid
flowchart TD
    REQ[Request] --> G{Free-tier declared,<br/>quota available,<br/>circuit closed?}
    G -- no --> N{Next provider?}
    G -- yes --> C[Call provider]
    C --> R{Result}
    R -- success --> OK[Return and cache]
    R -- transient --> RT{Retries left?<br/>max 3 requests per provider}
    RT -- yes --> C
    RT -- no --> N
    R -- rate limited --> BL[Wait if short, else block until reset] --> N
    R -- quota or paid-only --> BL
    R -- malformed output --> N
    N -- yes --> C
    N -- no --> P[Pause task with reason and resume time]
    P -.->|worker resumes later| REQ
```

Text chain: Gemini, then Groq, then pause. Transcription and moment discovery use Groq only, restricted to the provider the user
consented to.

## 6. Autonomous orchestration

```mermaid
flowchart TD
    TICK[Scheduler pass, every 15 s] --> CHK{Enabled, not paused,<br/>no blockers?}
    CHK -- no --> BLK[Blocked: reasons shown]
    CHK -- yes --> DISC[Discover from due sources]
    DISC --> SEL[Select by user rules<br/>each decision has a reason]
    SEL --> ACQ[Acquire permitted videos]
    ACQ --> IMP[Import job]
    IMP --> TR[Transcription job]
    TR --> MD[Moment discovery job]
    MD --> Q[Local quality filter]
    Q --> RND[Render jobs]
    RND --> REV[Review queue<br/>nothing is published]
    ACQ -- failure --> RP[retry_pending<br/>30 min back-off times attempt]
    RP --> ACQ
    ACQ -- 5 failures in a row --> PS[Autonomy pauses itself]
```

The scheduler creates ordinary jobs through the existing services; it does no media work itself. Only the discovery step was
observed running in this review.
