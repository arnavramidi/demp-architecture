# Decision log

The major calls, oldest first. Each entry has what I chose, what I considered, and what it cost me.

---

### v1 · Descriptor labeling instead of vector retrieve-and-rerank
- **Chose:** label each listing once at ingest from a fixed vocabulary, and match by set intersection with hard constraints as gates.
- **Instead of:** pgvector retrieval with a bi-encoder, then a cross-encoder rerank per user.
- **Why:** the rerank's cost grew with listings × users. Labeling costs per listing, and a new user adds no model calls.
- **Cost:** a fixed vocabulary can't express everything an embedding can, so matching is only as good as the descriptors.

### v1 · Learn each form once instead of self-healing
- **Chose:** a model maps the form once onto a declarative schema, which replays as a deterministic script.
- **Instead of:** a self-healing design.
- **Why:** self-healing failures couldn't be reproduced. Learning once cut LLM spend about 8×.
- **Cost:** a learned script can go stale silently. Answered by invalidating on failure, not on a timer.

### v1 · Its own queue for learning
- **Chose:** fill workers park jobs on unfamiliar platforms, and a dedicated learner builds each script once and republishes the jobs.
- **Instead of:** a lease-based design.
- **Why:** over 30% faster, and one user's new platform becomes every later user's cached one.

### v1 → v2 · Shut down the distributed version
- **Chose:** turn off v1 (June 30, 2026) and rebuild from the smallest possible base.
- **Why:** hosting cost was out of proportion to a free product with 25 users. The scale v1 was built for wasn't the scale it had.
- **Cost:** existing users lost the service while v2 caught up.

### v2 · No Kafka, no Redis: the database is the message bus
- **Chose:** one Postgres for jobs, work requests, leases and heartbeats.
- **Instead of:** Kafka for work, Redis for cache and locks.
- **Why:** a queue is a second system *beside* the database, not a replacement for it. A cache is where "one source of truth" dies. Revisit only with several consumers or a second stack.
- **Cost:** polling. v2 polled every second; v3 fixed that inside the same database with `LISTEN/NOTIFY`.

### v2 · Submit from the person's browser
- **Chose:** a Chrome extension sends the application from the person's own browser and connection.
- **Instead of:** submitting from the server that filled the form.
- **Why:** in a cloud test, submits from datacenter addresses were blocked 4 of 4 times. It also means the person sees exactly what's sent.
- **Cost:** a browser extension to build, ship and keep in step with the server.

### v2 · SERIALIZABLE for every multi-step write
- **Chose:** at the time, strict isolation with automatic retries.
- **Why:** SQLite had given "one writer at a time" for free, and I didn't want to lose that guarantee in the move to Postgres.
- **In hindsight:** too cautious. A queue claim doesn't need it, and with many workers it means retry storms. v3 scoped it down.

### v2 · Screenshots in the database
- **Chose:** at the time, PNGs in a table, so a backup is everything and there's one less vendor.
- **In hindsight:** screenshots were 78% of the database within two weeks of one person. v3 moved them to object storage.

### v2 → v3 · Don't ship v2 to Render
- **Chose:** redesign before hosting.
- **Why:** a global fill lock, one-second polling, files in rows, and one shared browser would all have moved to Render unchanged. Hosting fixes none of them.
- **Cost:** about two weeks of hosting groundwork set aside, though most of it (accounts, container image, AI through the API) carried into v3.

### v3 · Managed database, everything else on one VPS
- **Chose:** Render Postgres ($6/month, 3-day point-in-time recovery) plus one US VPS for web, source, match and workers.
- **Instead of:** everything on Render (~$240/month sized for Chrome), everything on AWS, or other managed databases ($15–$125/month with recovery).
- **Why:** the database is the one thing I want someone else to back up. Everything else is stateless, and Chrome needs memory that's cheap on a VPS.
- **Cost:** every page load crosses the internet to the database. The fallback, moving the web process next to it, is a configuration change.

### v3 · One table as the queue, claimed with an advisory lock and SKIP LOCKED
- **Chose:** a single work table for every kind of browser job, ordered by priority, then round-robin by person, then arrival, with a per-board cap counted in the database.
- **Why:** one order, one way to claim, one way to recover. Exact fairness and an exact cap across workers.
- **Cost:** claims are serialized, a few milliseconds each. That's fine at this scale and would be revisited long before it isn't.

### v3 · Lanes with an express lane
- **Chose:** two workers with three lanes each, one lane per worker taking only presses.
- **Why:** a person who presses Retry (a **press**) shouldn't wait behind a background fill.
- **Cost:** memory held in reserve for presses.

### v3 · Notifications sent inside the writing transaction
- **Chose:** `pg_notify` inside the same transaction as the change, with slow polling kept as a safety net.
- **Why:** listeners hear only about committed rows, and a dropped connection can't lose work, because the catch-up read and the safety-net poll pick it up.

### v3 · Isolation by operation
- **Chose:** `READ COMMITTED` plus explicit row locks for queue operations; `SERIALIZABLE` kept for application state, with every retried conflict logged.
- **Cost:** row locks are easy to forget. An eight-worker load test checks for retry storms; if it finds them, the fallback is a wrapper that always locks the application row first.

### v3 · Row-level security before the first outside user
- **Why:** "every query names its owner" works until someone forgets once, and an earlier multi-user branch was abandoned after review found six such places.

### v3 · Screenshot retention follows the application
- **Chose:** keep while unfinished, delete review screenshots 30 days after it's done, keep the confirmation screenshot of a sent application for the life of the account.
- **Instead of:** a flat N-day expiry, which would delete proof of applications people still care about.

### Learning · Check for an embedded known form before learning
- **Chose:** when a posting's URL isn't on a known board, inspect the page for an embedded form from an ATS that already has a playbook, run one trial fill with that playbook, and file the posting under it only if the trial is clean.
- **Instead of:** learning every unknown URL from scratch, or trusting the embed match without a trial.
- **Why:** many careers pages are a known ATS underneath, so recognizing one costs a page load, a replay and a cheap review instead of a full learning run. The trial is there because embedded forms are often tweaked.
- **Cost:** a failed trial is paid on top of the full learning that follows it.

### Learning · Cheapest first: no model, then a cheap reviewer, then the expensive learner
- **Chose:** try each new board with the generic filler and no model; have Claude Haiku approve or reject the attempt; call Claude Opus to learn the board only on rejection.
- **Instead of:** sending every unknown board straight to the expensive model.
- **Why:** plenty of forms are ordinary enough that no model needs to work them out, and the cheap review costs a fraction of a learning run.
- **Cost:** the cheap reviewer gates the cheapest path, so its misses matter. It never decides alone: deterministic read-back and page checks have to pass too.

### Every model call · Prompt caching
- **Chose:** put the parts of each prompt that don't change (instructions, tool definitions) first, so Claude's and Gemini's prompt caches cover them.
- **Why:** a learning run is many calls in a row with the same prefix. After the first, each call pays full price only for the new page snapshot.

### v3 · Learning in the shared queue, with a spending ceiling per board
- **Chose:** learn tasks at priority 20 in the same queue, one run per worker at a time, a cap on model turns per attempt, and three strikes before an admin is asked.
- **Why:** nobody's fill ever waits on learning, and no board can quietly eat the AI budget.
