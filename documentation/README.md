# Documentation

Technical documentation for AI Content Factory. Every statement was checked against the private repository (and, where
stated, a running copy) on 2026-11-25. Figures recorded earlier and not re-run are marked with their date.

| Document | Contents |
|---|---|
| [PRODUCT_OVERVIEW.md](PRODUCT_OVERVIEW.md) | Vision, problem, target users, capabilities, workflow, current status |
| [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md) | Frontend, backend, engine separation, database, job orchestration, communication, failure handling |
| [MEDIA_ACQUISITION.md](MEDIA_ACQUISITION.md) | Exactly what works today: acquisition architecture, provider abstraction, downloading, import, validation, retries, integration status |
| [PROCESSING_PIPELINE.md](PROCESSING_PIPELINE.md) | Ingestion, transcription, moment discovery, quality filtering, clip generation, FFmpeg, storage, review |
| [QUEUE_AND_ORCHESTRATION.md](QUEUE_AND_ORCHESTRATION.md) | Persistent queues, job lifecycle, pause/resume/cancel/retry, restart recovery, autonomous scheduling |
| [AI_PROVIDER_ARCHITECTURE.md](AI_PROVIDER_ARCHITECTURE.md) | Gemini and Groq, abstraction, fallback, circuit breakers, quotas, model use by stage |
| [TESTING_AND_RELIABILITY.md](TESTING_AND_RELIABILITY.md) | Testing architecture, backend and frontend tests, provider mocking, reliability testing, verified counts |
| [ENGINEERING_DECISIONS.md](ENGINEERING_DECISIONS.md) | Challenges, decisions, trade-offs, lessons |

Diagrams are in [`../architecture/`](../architecture/README.md) and screenshots in [`../screenshots/`](../screenshots/README.md).

**Status vocabulary.** *Verified live* means performed in the running application during this review. *Verified by tests*
means covered by an automated test run on 2026-11-25. *Implemented, not verified end to end* means the code exists and is
tested with test doubles but was never run against the real external service. *Under development* and *Not built* are used
literally.
