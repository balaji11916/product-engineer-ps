# Product Engineering Challenge Submission

## Candidate

- **Name:** Balaji E
- **Email:** balajielumalai6369@gmail.com
- **GitHub:** https://github.com/balaji11916
- **Selected problem:** Problem 3 — Durable Reminders and Follow-Ups
- **Demo video:** Pending recording and accessible link. See [DEMO.md](DEMO.md) for the 3–5 minute script.

**Submission status:** Implementation, automated tests, benchmark, and demo script are prepared. The demo video and candidate-verified credibility note are still required before final submission. This document does not claim either has been completed.

Specification: the unmodified [Problem 3 brief](problems/03-durable-reminders/README.md), repository revision `6c5d0c24e6ab55c6e23db05414c21360b318e2b3`.

## Run the project

Prerequisite: Python 3.11 or newer. No paid services, API keys, environment variables, database server, or Docker are needed. `tzdata` supplies IANA time zones on Windows; the rest uses Python's standard library. Run commands from the repository root.

Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m reminders.demo --step
```

macOS/Linux:

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m reminders.demo --step
```

The guided demo pauses for narration before each scene. Omit `--step` for an automatic smoke test. It uses a fresh temporary directory, actual subprocess restarts, and controlled time; it does not modify the normal `data/` directory. In the commands below, `python` means the interpreter in the virtual environment created above (or activate that environment first).

A manual successful scenario:

```sh
python -m reminders --data data/example --now 2026-09-20T03:29:00Z create interview --content "Prepare for interview" --local-time 2026-09-20T09:00:00 --zone Asia/Kolkata
python -m reminders --data data/example --now 2026-09-20T03:29:00Z tick
python -m reminders --data data/example --now 2026-09-20T03:30:00Z tick
python -m reminders --data data/example show interview
python -m reminders --data data/example inbox
```

The first tick executes zero jobs; the second delivers one. `show` returns state, version, ordered attempt history, and edit/cancellation history. The CLI renders instants in UTC ISO format. Repeating the same creation is idempotent; different data with an existing ID is rejected.

Temporary failure and recovery (use a new ID if re-running this example):

```sh
python -m reminders --data data/retry --now 2026-09-20T03:29:00Z create retry-example --content "Follow up on interview" --local-time 2026-09-20T09:00:00 --zone Asia/Kolkata
python -m reminders --data data/retry destination retry-example --mode temporary --failures 1
python -m reminders --data data/retry --now 2026-09-20T03:30:00Z tick
python -m reminders --data data/retry show retry-example
python -m reminders --data data/retry --now 2026-09-20T03:30:05Z tick
python -m reminders --data data/retry show retry-example
```

Other operations:

```sh
python -m reminders --data data/example list
python -m reminders --data data/example calls
python -m reminders --help
```

For an undelivered item, `edit ID --version 1 --content "New text" --local-time 2026-09-20T10:00:00` changes its content/time; `cancel ID --version 1` cancels it. Both require the current version shown by `show`. Delivered, failed, and cancelled items are terminal; create a new ID for new work. Repeating cancellation at the current version is harmless.

For real time, create an item with a future local date and omit `--now`. Run `python -m reminders worker` in one terminal, and use other commands in another. Stop with Ctrl+C; restart with the same `--data` directory. `worker` polls every second; `tick` does one discovery/execution pass and exits. Controlled-time examples intentionally use `tick`. Keep injected time moving forward across writes; `--now` is per invocation, not a persisted virtual clock.

## Run the tests

```sh
python -m unittest discover -s tests -v
```

Observed: **24 tests passed** on Python 3.12.14 / Windows. Tests use temporary SQLite files and an injected clock. Thread synchronization uses events with a timeout only to detect deadlocks; no correctness test depends on arbitrary sleeps or real minutes. The CLI test also runs separate processes.

## Acceptance scenarios and verification

| Scenario | Implementation and evidence |
| --- | --- |
| AC1 Scheduled delivery | Due queries use the injected clock; a job cannot be claimed before its instant. Success records a destination receipt and an attempt. |
| AC2 Restart recovery | All schedules and claims are durable. New service processes discover overdue items. Abandoned claims are recovered after a 30-second lease. |
| AC3 Temporary failure | Up to three attempts with five- then ten-second delays; permanent failures stop immediately. Tests check both recovery and exhaustion. |
| AC4 Duplicate execution | Source version/token fencing rejects stale execution. Destination primary key deduplicates repeated delivery calls, including after restart. |
| AC5 Edit before execution | Compare the expected version, increment it, invalidate the old claim, and retain the old attempt history. Tests edit a claimed job and prove only the new content arrives. |
| AC6 Cancellation | Cancellation that wins the source transaction prevents subsequent execution. If the destination already committed, cancellation is rejected and source state is reconciled to delivered. Both race orders are tested. |
| AC7 Time-zone boundary | Asia/Kolkata and America/New_York conversions, rejected spring DST gaps, and explicit choices for repeated autumn times are tested. |

Repeatable verification command:

```sh
python -m reminders.benchmark --output evidence/benchmark.json
```

The benchmark creates **24 items**: 8 normal, 4 edited, 4 cancelled, 4 temporarily failing, 2 permanently rejected, and 2 exhausting retries. It uses Asia/Kolkata and America/New_York, including the repeated New York local time on 1 November 2026. A seed process exits after four deliveries with one additional claim unfinished. A new process opens the same files, advances time, recovers that claim, and settles the work. It then invokes the destination twice more for one existing delivery key.

Observed results in [evidence/benchmark.json](evidence/benchmark.json):

- Before restart: 4 delivered, 4 cancelled, 1 running, 15 scheduled.
- After restart: **16 delivered, 4 cancelled, 4 failed**; no active work remains.
- **16 logical notifications**, exactly one for every delivered occurrence.
- Both repeated destination calls returned `deduplicated`.
- **28 recorded attempts**, including one interrupted claim recovered after restart.
- Zero unexpected delivery keys, superseded-content notifications, or count mismatches.
- The command exits nonzero when the checked expectations are violated.

The guided demo reproduces successful delivery, overdue recovery, temporary failure/retry, edit/cancellation, duplicate delivery, and the benchmark. The video itself has not yet been recorded.

## Architecture and data flow

```text
CLI create/edit/cancel/show
           |
           v
Service -----> reminders.sqlite3
  |            items + attempts + changes
  | claim due item using injected clock
  | commit claim/version/token and attempt
  | recheck claim under source writer lock
  v
LocalDestination -----> destination.sqlite3
  |                     notifications (unique delivery key)
  |                     provider behavior + call history
  v
Service records outcome / next retry / terminal state
```

- `clock.py`: system/manual clocks and strict local-time conversion.
- `database.py`: connection lifecycle, full synchronous durability, and transactions.
- `storage.py`: source schema and consistent inspection reads.
- `service.py`: lifecycle operations, claims, recovery, retry scheduling, and race policy.
- `destination.py`: separate durable local inbox, deduplication, receipts, and deterministic fault injection.
- `__main__.py`: CLI and optional polling worker.
- `benchmark.py` and `demo.py`: reproducible evidence and demonstration.

States: `scheduled -> running -> delivered`; temporary failure returns to `scheduled`; permanent failure/exhaustion becomes `failed`. `scheduled` or `running` may become `cancelled`; editing either creates the next version in `scheduled`. A stale worker cannot change the current version.

An attempt is persisted when a claim is committed, before the destination call. Thus a claimed-but-never-sent attempt remains visible as interrupted, superseded, or cancelled. History retains item/version/key, attempt number, start/finish time, outcome, and detail. The destination separately records each actual call. An unknown process failure is not mislabelled as a successful or failed HTTP request.

## Technology choices

Python is readable and directly supports SQLite, date/time handling, a CLI, subprocesses, and unit tests. SQLite keeps setup small and persistence inspectable. The local destination is explicitly permitted by the brief and needs no third-party credentials. `tzdata` is the only installed dependency.

I considered a web framework with PostgreSQL and a queue, but the required interface can be a CLI and this prototype does not need a distributed deployment. The accepted trade-off is serialized writes and limited throughput. Two database files deliberately expose the crash gap between the notification commit and the source acknowledgement; a single shared transaction would hide that failure boundary.

## Important decisions

1. **Explicit time semantics.** Accept a local ISO wall time and a named IANA zone; reject wall times containing offsets. Convert to UTC and retain the original time, zone, fold choice, and UTC instant. Nonexistent DST times are rejected. Repeated times require `--fold 0` for the first occurrence or `--fold 1` for the second. The stored instant remains fixed if time-zone rules later change; an explicit edit recomputes it. Past times are allowed and become immediately due.

2. **Durable occurrence identity and bounded retry.** A unique occurrence is `item_id + version`, e.g. `interview:v2`. Retries keep this key. Edits create a new key, retain history, and reset the attempt count. Due work is claimed with `BEGIN IMMEDIATE`, an incremented attempt number, a random claim token, and a 30-second lease. At most one current token is valid. Temporary destination unavailability is retryable; permanent rejection is terminal. Default policy: three total attempts, five then ten seconds of backoff. Interrupted claims count toward this limit; after expiry they are immediately eligible again unless exhausted. Policy parameters live in the Service constructor, and all workers must use the same configuration. No infinite retry is claimed, and successful delivery is not guaranteed during permanent failure.

3. **Idempotency and deterministic races at the delivery boundary.** A primary key on the destination receipt makes repeated attempts one logical notification. The destination commits its receipt and call history together. If acknowledgement is lost, lookup confirms the prior commit. Source edits/cancellation and delivery share a source writer lock; version/token checks reject stale claims. Whichever operation obtains and commits this lock first determines the outcome. If an edit/cancel commits first, the old claim cannot send. If the destination has already committed, edit/cancel reconciles delivered state and returns a conflict. A process crash after destination commit leaves a durable running attempt; receipt lookup on recovery prevents a second notification, even on the last allowed attempt.

This is at-least-once delivery *attempting with a finite retry budget*, plus one logical notification per successful occurrence at this local idempotent destination. It does not promise exactly-once external email/SMS delivery. An external provider would need equivalent durable idempotency and receipt/cancellation semantics.

## Assumptions and limitations

- One-shot reminders and conversational follow-ups use the same content field. Natural-language parsing, recurrence, real email/SMS/push, a dashboard, authentication, and multi-tenancy are deliberately absent, as permitted by the brief.
- One machine with persistent local disks; retain both database files together. Deleting destination receipts or restoring inconsistent backups invalidates deduplication/reconciliation guarantees. Keep receipt keys for at least the retry/replay lifetime.
- The local destination call occurs while holding the source writer lock. This makes races explainable and deterministic, but serializes delivery and user writes. It is appropriate only for a quick local destination, not a slow remote API.
- Multiple workers using these exact source/destination files and the same clock/policy serialize safely. This is not a multi-host solution; SQLite lock contention limits throughput. Workers must not bypass the Service path and call the sink with stale versions.
- A crashed claimed job can wait up to the remaining 30-second lease before recovery. Scheduled overdue work is processed at the next poll; no catch-up is possible while every worker is stopped.
- Unexpected software/storage exceptions propagate. A committed claim remains inspectable and recoverable; an operator should investigate repeated unexpected failures. Retry policy classifies the fake destination's explicit temporary/permanent errors, not arbitrary exceptions.
- No claim that the candidate has independently reviewed every line yet. Candidate review and practice are required before submission/interview.
- Remaining submission evidence: accessible narrated demo link and candidate-confirmed prior-work credibility note.

## Production and scale

The submitted implementation is local. For production I would first use a transactional database with per-row claims and fencing, preserve stable occurrence keys, and integrate a notification provider with durable idempotency and receipt lookup. I would not simply replace the local sink with an HTTP call while retaining the global database lock.

Cancellation at an external boundary needs a provider-supported conditional send/cancellation protocol or a clearly documented best-effort cutoff. A database edit alone cannot revoke a remote message that was already accepted. For many workers, I would add lease renewal, consistent clock/policy management, partitioned queues, and safe database migrations.

Bound retries, destination-specific concurrency limits, circuit breakers, retry jitter, and dead-letter inspection would keep an unhealthy provider from using all capacity. I would monitor queue depth, oldest overdue age, delivery lateness, success/failure rates, retries, exhausted jobs, expired leases, deduplication counts, and database lock latency. Alerts should cover sustained backlog, elevated terminal failures, and repeated lease expiry. Backups, retention, structured logs, authentication, and privacy controls would be required before real user data is accepted.

## AI usage

OpenAI ChatGPT helped discuss the assessment and choose a problem. OpenAI Codex read the official requirements, generated the implementation, automated tests, benchmark, demo script, and documentation, and ran local verification. The code is substantially AI-assisted; it is not presented as entirely manually authored.

Verification performed by Codex includes 24 behavior tests, separate-process CLI checks, the required 24-item benchmark, and the automatic demo. Balaji must review the code and explanations, record the demonstration, and be able to explain or modify the design before submitting. No unperformed candidate review is claimed.

## Credibility note

**Pending candidate confirmation.** Balaji has not yet provided enough information to accurately describe a previously shipped system, his personal contribution, its operational scale, or a difficult decision. No employment, usage figures, or achievements have been invented.

Before final submission, replace this paragraph with a short truthful account covering the problem, personal contribution, actual scale or complexity, one difficult decision, and a public link where available. A personal project is acceptable to describe honestly; distinguish building a prototype from operating a product used by others.
