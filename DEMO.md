# Demo recording guide (target: 4 minutes)

Use a terminal with readable text and the repository open in your editor. Run the project once before recording. Record only the project window; do not show your email inbox or credentials. This script is preparation, not an already-recorded video.

Start from the repository root using the virtual-environment Python:

```sh
python -m reminders.demo --step
```

Press Enter at each scene. The script creates temporary databases, executes the real CLI in separate processes, and prints checked outcomes. No real-time waiting is required.

| Time | Show | Suggested narration |
| --- | --- | --- |
| 0:00–0:25 | Problem 3 and start the demo | “This is a Python service for one-shot reminders and follow-ups. I focused on durable scheduling, bounded retries, safe edits, and preventing duplicate logical notifications. AI assisted the implementation, and I will explain the decisions.” |
| 0:25–0:55 | Scene 1 | “This reminder is due at 9 AM in Asia/Kolkata, which is 03:30 UTC. Before that instant, nothing runs. Advancing the injected clock delivers one notification and records the attempt.” |
| 0:55–1:20 | Scene 2 | “The creation process exits. A later process opens the same SQLite files after the due time. It finds and delivers the overdue reminder. Schedules survive process restarts.” |
| 1:20–1:55 | Scene 3 | “The local destination fails once. The first attempt is visible in history. After five seconds on the test clock, the retry succeeds. The default policy permits three attempts, with five- and ten-second delays.” |
| 1:55–2:25 | Scene 4 | “An edit creates version 2 and invalidates any old claim. A cancellation committed before delivery prevents sending. Once delivery commits, the service reports that it is too late to cancel.” |
| 2:25–2:50 | Scene 5 | “These two calls repeat the same delivery key. The destination stores that key uniquely, so both calls are deduplicated and the logical notification count stays one.” |
| 2:50–3:20 | Scene 6 | “The required benchmark runs 24 items across two time zones, including a daylight-saving boundary. It restarts the process with unfinished work and settles at 16 delivered, four cancelled, and four failed as intended. There is one notification per successful occurrence.” |
| 3:20–3:50 | SUBMISSION architecture diagram and service.py | “The CLI calls the service. SQLite stores schedules, versions and attempts. A separate SQLite destination stores notifications and receipt keys. A claim is committed before delivery; after a crash, receipts let us reconcile an accepted notification.” |
| 3:50–4:15 | Tests and limitations | “The trade-off is holding a source write lock during the fast local delivery. It makes cancellation races deterministic but limits concurrency. A remote provider would need a different boundary protocol. The 24 automated tests cover these behaviors.” |

Avoid claiming you wrote every line without assistance or have operated this at production scale. Only say you understand a part after you have studied and tried it. You can pause between scenes and explain in your own words.

After recording:

1. Upload the 3–5 minute recording to a service you use.
2. Enable reviewer access to the video and test the link outside your account.
3. Put its actual URL near the top of SUBMISSION.md.
4. Complete the credibility note with your real experience.
5. Submit the repository through the form linked in the official README. The recruitment email specifically requires the form.
