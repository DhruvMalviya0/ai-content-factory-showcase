# Testing and reliability

How AI Content Factory is tested, which numbers were verified for this repository, and what the tests do not cover.

## 1. Verified results

Both suites were run on **2026-11-25** from an **isolated copy** of the private repository's working tree, with a fresh
`.env` containing no API keys.

| Suite | Result |
|---|---|
| **Dashboard unit tests** (Vitest) | **22 files, 406 tests, all passed** (about 13 seconds) |
| **Backend suite** (pytest, `REQUIRE_FFMPEG=1` so the real-FFmpeg tests cannot silently skip) | **1,846 tests collected: 1,843 passed, 3 skipped, 0 failed.** The three skips are environment-dependent: two need permission to create symbolic links or junctions, and one tests a memory cap that was not enforceable in the run environment. |

How to read the counts:

- The backend suite defines **1,031 test functions in 46 files**. Pytest collects **1,846 tests** because many functions are
  parametrised over several cases.
- The backend working tree contained uncommitted changes when it was copied (the private repository had 26 modified files),
  so these results describe the working tree, not a tagged release.
- The backend suite was run as **four parallel batches** of whole test files (the full run is slow because the flow tests use
  real FFmpeg), each in its own process with its own temporary data. The four batch results add up to the full 1,846.
- Nothing contacted a real provider, website or external service during either run.

## 2. Testing architecture

| Layer | Tooling | What it exercises |
|---|---|---|
| Backend unit tests | pytest | Pure logic: moment validation, render planning, scoring, path handling, circuit breaker, planner |
| Backend integration and flow tests | pytest with the FastAPI test client, real SQLite databases in temp folders, real FFmpeg on tiny generated footage | Whole flows through the real API, queue and worker: import, transcription, discovery, rendering, review, autonomy, acquisition |
| Migration tests | pytest on temporary databases | Upgrade and downgrade of the phase migrations, never the live database |
| Dashboard unit and component tests | Vitest, Testing Library | State machines, formatting, forms, panels, the API proxy and its security headers |
| End-to-end tests | Playwright | Two spec files (10 and 4 tests) against **isolated** backends and dashboards on their own ports, described below |
| Script checks | PowerShell | A parser check that every script parses under Windows PowerShell 5.1, and a desktop-to-laptop portability simulation on temporary data |

### Backend tests by area (test functions defined)

| Area | Functions |
|---|---:|
| Provider layer (manager, adapters, circuit breaker, half-open probe, API, verification audit) | 126 |
| Media job queue, process runner, hardening and observability | 125 |
| Import and media library (library, probe, paths, API, migration) | 116 |
| Moment discovery (flow, engine, migration) | 98 |
| Clip rendering and review (plan, flows, review, retry race, migration) | 95 |
| Transcription (flow, units, audio, migration) | 94 |
| Source acquisition (units, flow, migration) | 72 |
| Security, authentication and fetching (hardening, host guard, auth, SSRF, robots.txt) | 111 |
| Campaigns | 59 |
| Platform (API, configuration and database, hardware and FFmpeg profiles) | 48 |
| Research, claim verification and scripts | 47 |
| Autonomous mode (units, flow) | 40 |
| **Total** | **1,031** |

These are counts of functions per area, not coverage measurements.

## 3. Provider mocking

The suite is designed so that **a real provider can never be reached**:

- A **fake provider** implements the text, speech and transcription interfaces and answers from a script or a small
  deterministic rule. A scripted "language model" plays the moment-discovery model, and the tests assert on the exact requests
  it received (for example that only transcript text and no path or file name was sent).
- A test fixture **blocks all non-loopback network access** by guarding socket connection and name resolution, so a
  forgotten mock fails loudly instead of calling out.
- The two vendor SDKs are exercised **for real over a mock HTTP transport**, to confirm which endpoints are called, that
  SDK-level retries are disabled, and that representative error bodies map to the right error class. This is the one place
  where the actual SDK code runs; it still never leaves the machine.
- A deterministic **offline provider** exists for the end-to-end suite and is refused outside the test environment.
- Tests scan responses and database columns for the fake key and for local directories to prove secrets and absolute paths
  never leak.

> **Consequence.** The provider integrations are verified against doubles and the SDKs' own exception classes. No real
> provider request has been made by the project (per its own verification record of 2026-11-17) or in this review.

## 4. Integration testing

- Flow tests import real, generated footage (a few kilobytes, from FFmpeg's test sources) through the real API, run the real
  queue worker, and render with the real encoder, then assert on the produced file with ffprobe. A control test confirms an
  unusual input (odd dimensions, rotation metadata, variable frame rate, no audio) is handled.
- Only the transcription and discovery *providers* are replaced; everything else in those flows is real.
- Playwright runs the dashboard against a backend in two isolated configurations: one with every API key forced blank, to prove
  **nothing is ever sent without a provider**, and one with the offline provider to exercise research, script and review.
  Both use throw-away data folders that are wiped on each run, so the real database, first-run token and backups are never
  touched.

## 5. Reliability testing

| Property tested | How |
|---|---|
| Child processes die with the backend | A test terminates the host process without cleanup and checks that FFmpeg is gone (a control without the job object leaves it alive). Windows only. |
| Restart and crash recovery | Tests re-queue interrupted jobs, finish interrupted cancels, reap orphans only on an identity match, and settle stranded rows |
| Concurrency | Race tests for simultaneous clip retries, cancel versus outcome, and claim versus cancel |
| Circuit breaker | Opening, doubling cooldown, the single half-open probe, token ownership, late releases, and zero-cooldown |
| Retry accounting | Separate provider-level retries and task-level attempts; pauses that never consume attempts |
| Idempotency | Duplicate submissions, duplicate content, and a render whose database save failed and is adopted on retry |
| Migrations | Upgrade and downgrade on temporary databases |
| Backup and restore | Consistent backups, checksum validation, and restore of data held only in the write-ahead log |
| Observability | Stuck-job detection that only reports, queue age, failure categories |
| Security hardening | Forwarding headers ignored, security headers on every response, Host-header guard, SSRF guard including mixed DNS answers and unusual IP spellings |

## 6. Frontend tests

The dashboard's 22 Vitest files cover: status and state mappings for acquisition, autonomy, campaigns, clips, media,
moments and transcripts; formatting; the campaign form; the moments, transcript, clips, clip-review and source-link panels;
the sidebar; the pages' rendering; and the API proxy's guards and security headers (including a dedicated route-security
suite). The Playwright suite adds 14 end-to-end tests. **The end-to-end suite, type check and lint were not run for this
repository.**

## 7. Test coverage boundaries

- **No coverage percentage is measured or claimed.**
- **No live provider calls**: transcription, moment discovery, research and scripting are verified with doubles.
- **No real-network acquisition**: downloads, feeds and YouTube metadata are verified against mocks.
- **No performance, load or scalability testing.** Nothing in this repository claims throughput.
- **Windows-specific protections** (job object) can only be exercised on Windows.
- **Single machine only**: the portability simulation covers moving between computers via backup and restore, not concurrent
  use.
- **Validation is metadata-level**: a corrupted video with intact headers is not detected by import (see
  [MEDIA_ACQUISITION.md](MEDIA_ACQUISITION.md#7-media-validation-and-integrity)).
- **Not run in this review:** the Playwright suite, ESLint, the type checker, and the PowerShell portability and parser checks.

## 8. Reliability claims this repository does not make

It does not claim production readiness, scalability, uptime, accuracy of AI-proposed moments, or that clips will perform well
when published. The scores are rankings within one video produced by a model's judgement; the quality filter removes obviously
weak clips and does not predict views.
