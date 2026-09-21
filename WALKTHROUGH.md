# Understand the project before the interview

Start with the real use case: “Remind me to prepare for my interview on 20 September at 9 AM in India.” The program must save that promise and handle interruptions without silently losing or duplicating it.

## The components in everyday terms

| Term | Meaning here | File |
| --- | --- | --- |
| CLI | Commands used to create, inspect, edit and cancel reminders | reminders/__main__.py |
| Database | A durable notebook saved on disk | reminders/storage.py |
| Scheduler | Finds reminders whose time has arrived | Service.claim_due in reminders/service.py |
| Worker | Claims a reminder and performs the delivery | Service.execute and Service.run_due |
| Destination | A local notification inbox that remembers received delivery keys | reminders/destination.py |
| Injected clock | A replaceable clock used to make tests fast and repeatable | reminders/clock.py |

## One successful reminder, step by step

1. You supply an ID, content, local date/time, and time zone.
2. The service validates the input and converts the time to UTC.
3. SQLite saves it as version 1, state `scheduled`, with key `interview:v1`.
4. The scheduler sees that it is due and claims it. A claim means a worker has permission to process this version. The running attempt is saved before sending.
5. The worker checks that its version and claim token are still current.
6. The local destination saves a notification under the unique delivery key.
7. The service records the attempt result and sets state `delivered`.

The scheduled time, current status, attempt history, and change history remain inspectable.

## Failures to understand

**Program stopped before the due time:** Nothing can be delivered while it is stopped. On restart, it reads saved schedules and catches up on overdue work.

**Worker died after claiming:** The claim has a 30-second lease. When that lease expires, recovery marks the attempt interrupted and makes the reminder available again, within the attempt limit. An old worker's token cannot overwrite a newer claim.

**Temporary destination failure:** Retry after five seconds, then ten seconds, for three total attempts. After the third failure, show `failed`. This prevents endless work.

**Permanent rejection:** Stop immediately; retrying the same rejected item would not help.

**Destination accepted, but source did not record success:** The two database files make this a real failure boundary. Recovery looks up the stable delivery key at the destination. An existing receipt means it already happened. Repeated destination calls return the existing logical result.

## Idempotency is not a magic “send only once” switch

The scheduler or caller can repeat an operation. The destination recognizes the same key and avoids another notification. That is the key guarantee. A provider that does not implement durable deduplication would need another design; our local result does not prove exactly-once email delivery.

## The editing race an interviewer may ask about

A worker holds version 1, scheduled for 9 AM. You edit to 10 AM, producing version 2.

- If the edit commits first, version 1 is superseded. Its worker sees the mismatch and does not send.
- If delivery gets the shared source lock first and commits to the destination, the edit is too late and is rejected.
- If the process crashes after sending, editing first checks the destination receipt and still reports that delivery already happened.

The same rule applies to cancellation. Saying “cancelled” after an irreversible notification has already committed would be misleading.

## Time zones

9 AM on 20 September 2026 in Asia/Kolkata equals 03:30 UTC. Using UTC internally lets due-work queries compare instants consistently. We also keep the local request and named time zone for inspection.

On 1 November 2026, 1:30 AM happens twice in America/New_York. `fold=0` chooses the first, 05:30 UTC; `fold=1` chooses the second, 06:30 UTC. The caller must choose. A spring-forward local time that never occurs is rejected rather than silently moved.

## Read the code in this order

1. `clock.py`: how time is supplied and converted.
2. `storage.py`: what gets saved.
3. `service.py`: create, claim_due, execute, then edit/cancel and recovery.
4. `destination.py`: the unique receipt key and failure simulation.
5. `tests/test_workflow.py`: read one test for each example above.

Run `python -m reminders.demo --step` and predict the result before each scene. Then change the temporary failure count from one to three and explain why the final state changes.

## Short interview introduction

“This is a Python service for reliable reminders. SQLite stores schedules and attempt history. A worker discovers due reminders, retries temporary failures within a fixed limit, and uses versions to prevent stale edits or workers from sending outdated content. A durable delivery key lets the local destination deduplicate repeated attempts. I used a controlled clock to test recovery and time zones without waiting in real time.”

## Honest trade-off to discuss

The source writer lock remains held during the quick local destination call. This simplifies race handling, but means delivery and edits are serialized. It is intentionally a small local prototype. For slow external delivery, I would use provider-side idempotency and an explicit cancellation/commit protocol instead of keeping a global database lock across a network call.

AI helped implement this project. Be open about that and practice explaining the design; do not memorize wording you cannot connect to the code.
