# Peer Stress Test

**Question:** can your partner run the runbook without asking you?

Swap manuals with another team. The partner follows only the written steps (runbook.md and api-documentation.md), on a test environment, while the author watches and does not help. The author only writes down every question the partner has to ask.

| # | What the partner must do from the manual alone | Done without help (Yes / No) | Question asked or confusing part |
| --- | --- | --- | --- |
| 1 | Find the API's status and confirm it is the problem (step 1) | Yes | None |
| 2 | Save the logs (step 2) | Yes | None |
| 3 | Restart only the Booking API, without touching the database or queue (step 3) | Yes | None |
| 4 | Confirm the API is healthy again (step 4) | Yes | None |
| 5 | Confirm the queue consumer is reconnected (step 5) | Yes | None |
| 6 | Run the smoke test and read the result (step 6) | Yes | None |
| 7 | Say what to do if it still fails (step 7) | Yes | None |
| 8 | Use the endpoint doc to send the sample request and understand a 401 or 409 | Yes | None |

**Pass:** all 8 rows are Yes and the partner asked the author 0 questions.
**Fail:** any question asked or any row marked No. Fix the unclear step in the manual and test again.

**Result:** PASS. Number of questions asked: 0. Steps fixed in the manual after the test: none needed.