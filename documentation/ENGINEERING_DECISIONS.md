# Engineering decisions

The main engineering challenges, decisions and trade-offs in AI Content Factory. Each entry is grounded in the private
repository and its design records (inspected 2026-11-25). Where an outcome has not been measured or run live, that is said.

## 1. Two queues, not one

**Decision.** Provider calls run on an AI task queue; local FFmpeg and file work runs on a separate media job queue with its
own single worker. AI calls never run on the media worker.

**Why.** The two have different failure modes. AI work fails by rate limit, quota and outage, and should *pause and resume*.
Media work fails by crash, disk space and corrupt files, and needs process supervision and recovery. Mixing them would make a
quota wait block a render, or a hung FFmpeg stall research.

**Trade-off.** Two mechanisms to maintain, and no shared Jobs view yet (status appears inside the footage and clip panels).

## 2. Enforce invariants in the database, not only in code

**Decision.** Correctness rules are check constraints and partial unique indexes: a media file can be *ready* only if its hash,
size, duration, resolution and codec are all recorded; content hash is unique among ready files; one active render per
candidate and profile; one active discovery run per footage; an unrecorded half-import cannot be marked complete.

**Why.** Code paths multiply (API, retries, recovery, the scheduler) and race. A rule the database refuses to break holds for
every writer, including future ones and crash recovery.

**Trade-off.** More schema work per feature, and migrations that must be careful (twelve so far, with a documented "back up
first" step because migrations run automatically at start).

## 3. Pause, never fail, on provider limits; two retry budgets

**Decision.** Rate limits, exhausted quotas, an open circuit breaker or an unconfirmed free tier *pause* a task until the earliest
known resume time. Only malformed output and unexpected errors consume a limited attempt budget. Provider-level retries and
task-level attempts are deliberately separate counters.

**Why.** Free-tier operation means hitting limits is normal, not exceptional. Treating it as failure would lose work and
exhaust retries; merging the two budgets would let provider fallback silently refill the task's attempts.

## 4. Code verifies claims; the model only proposes them

**Decision.** A research claim is stored with its source and a verbatim quote, and **code** checks the quote against the
retrieved text. Statements whose meaning may have drifted from the quote (metric, percent versus percentage points, dropped
time periods, removed hedging, causation, scope, superlatives, negation) go to `needs_review`; health, finance, legal and
death-related claims always do. Only verified or human-approved facts are offered to the script generator, and a script is
checked deterministically before it can be approved.

**Trade-off.** Fewer usable facts per topic. The documentation is explicit that string matching proves evidence exists, not that
the statement interprets it correctly, so human review remains the final safeguard.

## 5. The model never writes timestamps

**Decision.** Moment discovery asks the model for the first and last *segment number* of a moment; times are then read from the
stored transcript plus a small pad.

**Why.** A model that writes timestamps can invent them. With segment numbers, a moment cannot start or end where nobody speaks,
and an out-of-range timestamp cannot exist. Local validation then **drops** bad suggestions rather than repairing them, and never
trims an over-long moment because that could cut the payoff.

## 6. Re-encode every cut; validate the output against the plan

**Decision.** Clips are always re-encoded, never stream-copied, and a zero exit code is never trusted: the staged file is probed
and must match the planned size, codecs, frame rate, duration and audio presence before it is renamed into the library.

**Why.** A stream copy can only start on a keyframe, which gives clips that start seconds early with undecodable lead-in frames.
A "successful" encode can still be wrong. **Trade-off:** re-encoding costs CPU time; CPU encoding was chosen deliberately for a
reliable baseline on a laptop-class machine.

## 7. Treat child processes as a liability on Windows

**Decision.** FFmpeg runs under a Windows job object with kill-on-close, with wall and stall timeouts, a graceful stop then a
whole-tree kill, a pid-start-time-executable identity check before reaping orphans, and a refusal to run at all if containment
cannot be established.

**Why.** A crashed backend that leaves an encoder running can fill a disk or corrupt a later run. A test terminates the host
without cleanup and verifies that the child dies with it.

## 8. Explicit ownership and race-safe transitions

**Challenge.** A post-implementation audit found concurrency defects, for example two simultaneous retries that both re-queued
the same job so the loser then cancelled the winner's work.

**Decision.** Every state change (claim, cancel, pause, resume, retry, outcome) is a conditional update with the row count
checked; a worker identity (machine, process, start time) establishes ownership; a worker that loses ownership is told to stop
and its late result is ignored; of several simultaneous retries exactly one wins.

**Lesson.** Concurrency bugs in queues are usually unconditional writes. Make every transition conditional on the state that was
read.

## 9. Import folder instead of browser upload

**Decision.** The API accepts *names of files inside an import folder*, never paths, and the dashboard offers a picker over that
folder. Files are copied (never moved) into a content-addressed library.

**Why.** Footage is often many gigabytes; the dashboard's proxy buffers request bodies, so browser upload is neither safe nor
practical. Names-not-paths removes path traversal. Content addressing makes duplicates impossible and crash repair trivial.

**Trade-off.** A second copy on disk and no drag-and-drop.

## 10. Do not download what the platform does not permit

**Decision.** There is no YouTube downloader. A pasted link yields validated metadata through the official Data API, and the
request waits for a video file the user supplies. The project decided earlier not to add a downloading tool, and no stream
extraction, cookie use or circumvention exists anywhere.

**A real example.** Autonomous YouTube discovery first used the public channel feed. A live check on 2026-11-21 showed that the
platform's `robots.txt` disallows that feed for every user agent, and the project honours `robots.txt`, so discovery moved to the
official API instead.

**Trade-off.** The product cannot "clip any YouTube link". The documentation says so rather than implying otherwise, and the
provider contract means a platform that does permit downloading needs no change to the pipeline.

## 11. Rights and consent are separate from capability

**Decision.** Three questions are recorded independently: *am I authorised to use this footage* (a statement), *is there a
permitted way to get the file* (decided by the adapter), and *may audio or text be sent to an AI provider* (a revocable consent
given per footage or per source). Consent is enforced in the backend three times (at request, at job start and before every
chunk) and withdrawing it cancels queued work.

**Why.** Conflating them is how footage gets uploaded that nobody agreed to send. A test pins that the import-time rights
tick is never treated as an upload permission.

## 12. Reuse the pipeline instead of building a second one

**Decision.** Autonomous mode and the YouTube-link request add *orchestration* only: every step is an ordinary job created
through the existing services. A video is recorded once per source; identical bytes reuse the existing transcript, moments and
clips.

**Why.** One import path, one set of protections, one place to fix bugs. **Trade-off:** autonomous mode inherits the queue's
single-worker throughput.

## 13. Localhost-only by design, with explicit trust decisions

**Decision.** Both services bind to the loopback address; a Host-header allow-list blocks DNS rebinding; the backend never trusts
forwarding headers (a hardening pass found the client address was spoofable through inherited server defaults and made the trust
decision explicit); security headers are applied by outermost middleware so even error responses carry them.

**Trade-off.** No LAN or phone access and no supported reverse-proxy deployment.

## 14. Honesty about free-tier status

**Decision.** "Zero paid API mode" calls a provider only if the user declared its key free-tier, and the interface labels that
"declared by you (unverified)", because neither provider exposes billing status. The real safeguard (billing disabled on the
project that owns the key) is stated plainly. Paid usage is not implemented.

## 15. Observed in this review: gaps worth recording

- The Overview's "Rendering" card still reads *"Idle — renderer arrives in Phase 5"* and the sidebar footer reads
  *"Phase 4B · Media library"*, although rendering exists. These are out-of-date interface labels, not functional defects.
- Schema tables for voiceovers, media assets, QC reports, publishing jobs and analytics exist with no implementation behind them.
- No live provider call has been made, so the provider-dependent half of the pipeline is verified with fakes only.

## 16. Lessons learned

- Decide what is *state* and what is *failure*: missing consent, a quota wait and "nothing worth clipping" are explainable states.
- Never trust an exit code; validate the artefact.
- Put the rule where every writer must obey it (the database), and make every transition conditional.
- Name your assumptions in the product: "estimated", "declared by you", "synthetic", "not a prediction of views".
- Write the limits down as carefully as the features.
