# How I Broke My Flash Sale Backend (And Fixed It): A Yoink! Case Study

I built a Node.js e-commerce backend called Yoink! to simulate what happens during a massive flash sale. The goal was simple: handle 1,000 concurrent users trying to buy a product at the exact same millisecond without overselling.

To do this right, I knew I needed robust testing and observability. I used **k6** to simulate the concurrent load (Virtual Users or VUs), **Prometheus** to scrape application metrics, and **Grafana** to visualize everything. 

It sounds easy until you actually load test it. I learned the hard way that a system working perfectly for five users will completely collapse under a spike. Here is the step-by-step evolution of my architecture, the infrastructure hurdles I faced, and why I had to keep rebuilding.

## Early Infrastructure Hurdles

Before I even got to the race conditions, I had to figure out how to observe them. This taught me a lot about networking and middleware:

**The Docker Firewall Trap:** I deployed my app and Prometheus in Docker containers, but Prometheus couldn't reach my `/metrics` endpoint. After hours of scanning logs, I finally checked `ufw` and `nftables` rules. The host firewall was actively blocking connections from the `docker-compose` network. I had to add the Docker subnet as an exception in `ufw`. This was a major "aha" moment for me regarding Docker—I learned exactly how Docker network isolation works under the hood and why you shouldn't just lazily bind everything to the host network.

**The Case of the Missing Metrics:** Once I got Prometheus scraping, I noticed a frustrating issue in Grafana. When a request threw an error and hit my Express global error handler, the origin path was erased. All my errors were being lumped together as untagged 500s or 400s; I couldn't tell which route was failing. I fixed this by writing a custom metrics middleware that explicitly saved the original path to the request object *before* it hit any controllers, ensuring Prometheus could properly tag the route even if the request exploded later.

With observability finally in place, I started load testing.

---

## V1: The Naive Implementation

<img width="2000" height="624" alt="image" src="https://github.com/user-attachments/assets/d06b4058-789d-4de4-98ec-0b15c7dfdec3" />

**What I built:** I started with the standard CRUD logic every developer learns first. Read the stock from Postgres, check if it's greater than 0, decrement the stock, and insert an order record. The architecture is dead simple: every request hits the Node.js server, which talks directly to Postgres with separate read and write queries.

**What broke:** I ran a k6 benchmark firing 1,000 concurrent users at the buy endpoint simultaneously. Before showing you the results, here's what the database looked like going in — 5,000 units of stock:

<img width="958" height="218" alt="Database before testing: product has 5000 stock" src="https://github.com/user-attachments/assets/20c060ab-c929-4f71-b9db-ddd06bfeffe3" />

The load test ran, and Grafana showed me just how much traffic the server was absorbing. This is the total confirmed orders over the test duration:

<img width="1864" height="903" alt="Grafana: total orders processed during V1 test — confirmed orders climbed far beyond the available 5000 stock" src="https://github.com/user-attachments/assets/55165ef6-f8ca-4823-99ad-1963e1b3c57b" />

The order count kept climbing — well past 5,000. But why? This graph shows what was happening under the hood. The p95 latency spiked and then 500 errors began flooding in:

<img width="1861" height="906" alt="Grafana: p95 latency — high latency followed by a wave of 500 errors" src="https://github.com/user-attachments/assets/b0f62595-30f2-4805-a3f7-980894dd22db" />

The errors weren't random — they had a specific cause. The database connection pool was being completely exhausted:

<img width="950" height="694" alt="Grafana: 500 errors with reason — connection pool exhaustion (too many clients)" src="https://github.com/user-attachments/assets/9a599da8-661a-4943-b136-55005791e805" />

With 1,000 requests hammering the server simultaneously, the Postgres connection pool ran out of available connections. The errors showing up in the logs were Prisma's `P2028` — technically a "Transaction API error," but in this context a direct symptom of pool exhaustion: transactions couldn't be opened because there were no free connections to acquire. Requests that couldn't get a connection failed with 500s, while the ones that *did* get through raced each other to read and write the stock value. This is the API throughput during that chaos:

<img width="1867" height="907" alt="Grafana: API requests per second during V1 — high initial throughput collapsing as the connection pool died" src="https://github.com/user-attachments/assets/5d433e3d-e768-4f2f-b062-cfbdb9570011" />

The confirmation of the damage came after the test. 5,000 units of stock, but the database now showed 2,851 remaining — and the order table had **28,486 confirmed orders**:

<img width="955" height="216" alt="Database after testing: 2851 stock remaining, but 28000+ orders were confirmed — catastrophic overselling" src="https://github.com/user-attachments/assets/7900de36-b944-4ece-a062-ed7f537e9c95" />

This is a classic **read-modify-write race condition**. Thousands of requests hit the database simultaneously, each read the stock as `> 0`, and each fired off a decrement. Postgres had no coordination between those concurrent reads — by the time any single request wrote its decrement, dozens of others had already read the same stale value and decided it was safe to proceed.

The full k6 summary confirms the extent of the failure:

<img width="1891" height="1141" alt="k6 test results for V1: high request counts, significant error rates, and no meaningful rate limiting" src="https://github.com/user-attachments/assets/8a12e6bf-8441-47fa-bc74-f93132ad9f0a" />

**What I learned:** Correctness under concurrency requires atomicity. If you read a value and write it back in separate, unisolated operations, you *will* oversell under load.

---

## V2: Atomic DB Transactions

<img width="2000" height="777" alt="image" src="https://github.com/user-attachments/assets/02fd3ee2-5e59-4540-b5f5-b508cd4590ad" />

**What I built:** I pushed the concurrency control down to the database. I swapped the separate read and write queries for a single Prisma `updateMany` operation with a `WHERE stock >= 1` clause. This enforces atomicity at the DB engine level — the read and decrement happen in one indivisible operation, so there's no window for another request to sneak in between them.

**Did it fix overselling?** Yes. Here's the database before the test — again 5,000 units:

<img width="955" height="216" alt="Database before V2 testing: 5000 stock" src="https://github.com/user-attachments/assets/ee2243e4-6afd-47b5-bf18-a0840a9b6395" />

And after the test, stock hit exactly 0, with exactly 5,000 confirmed orders. No more overselling:

<img width="961" height="219" alt="Database after V2 testing: 0 stock remaining, exactly 5000 orders — overselling is solved" src="https://github.com/user-attachments/assets/4183b779-371a-40f3-a797-0ffb05ecb096" />

**But a new problem emerged — performance.** Look at the total orders graph. There's a large spike of failed orders alongside the 5,000 successful ones:

<img width="1846" height="905" alt="Grafana: total orders during V2 — 5000 successful orders and a large volume of failures" src="https://github.com/user-attachments/assets/3ab04244-eace-45c4-a257-9e99cb48cad3" />

Those failures aren't bugs — they're expected "out of stock" rejections as the inventory ran out. This graph confirms it: failed orders (404s) spiked exactly as 500 errors disappeared, meaning the server was correctly refusing requests once stock hit 0:

<img width="1858" height="904" alt="Grafana: error breakdown — 404 out-of-stock errors replaced 500 connection errors, confirming correct behavior" src="https://github.com/user-attachments/assets/91f7254f-f189-4c63-9701-c46faad48bc9" />

The real cost shows up in latency. When stock was still available and 1,000 users were fighting for a DB connection to run that atomic update, contention was extreme. The p95 latency peaked above **476ms** — and ironically, it only dropped *because* the stock ran out and the server stopped hitting the database entirely for out-of-stock requests:

<img width="1851" height="911" alt="Grafana: p95 latency in V2 — latency drops sharply once stock is exhausted and DB writes stop" src="https://github.com/user-attachments/assets/2ff1ab24-444a-4b1c-b53c-924011ac5829" />

The throughput tells the same story — the server handles the burst, but it's clearly straining:

<img width="1856" height="903" alt="Grafana: API requests per second during V2 — throughput held but latency was high during the inventory window" src="https://github.com/user-attachments/assets/678b1178-bbf8-46c5-a4d0-10151ca2336f" />

The k6 summary confirms the tradeoff: correctness achieved, but at significant latency cost during peak contention:
<img width="1886" height="1128" alt="image" src="https://github.com/user-attachments/assets/d2351ddb-7d66-4174-9e21-737aad1ee94f" />

**What I learned:** Correctness isn't enough. A correct system that can't handle load isn't production-ready. The database is the bottleneck — and pushing harder on it just makes things worse.

---

## V3 (Initial): Redis + Lua

<img width="2000" height="679" alt="V3 Initial Architecture: Client → Node.js → Redis (Lua atomic decrement) → if success → synchronous Postgres write" src="https://github.com/user-attachments/assets/1764419f-4e15-4557-a744-a649bb523acd" />

**What I built:** Postgres was the bottleneck in V2, so I moved inventory management out of it entirely. Instead of asking Postgres to coordinate concurrent decrements under load, I wrote a Lua script that atomically decrements the stock counter directly in Redis. Redis processes operations sequentially on a single thread — no race conditions, no lock contention, around 10,000 ops/second. Once the Lua script confirmed a successful decrement, the service made a synchronous write to Postgres to persist the order record.

The split of responsibilities: Redis owns the fast, conflict-free inventory check. Postgres just stores the receipts.

**A false sense of success**

My first serious test was 1,000 VUs against a flash sale with scarce stock — a few thousand units. It *worked*. No 500s, clean error rates, latency well below 500ms. I thought I'd finally solved it.

But there was a hidden reason it worked, and it had nothing to do with the architecture being correct. When stock ran out, the Lua script returned early and the server sent a 404 *without ever touching Postgres*. With scarce stock, the vast majority of requests hit the out-of-stock path and bypassed the database entirely — so the connection pool never got overwhelmed. The test looked healthy because most traffic never reached the bottleneck.

**What actually broke**

As soon as I bumped inventory to 10,000,000 units — enough that a large proportion of requests would succeed and each need a DB write — the system fell apart. Prisma started throwing `P2028`:

```text
Transaction API error: sorry, too many clients already
```

Redis was clearing the inventory check so fast that all 1,000 requests arrived at the Postgres write simultaneously. The connection pool had nowhere near enough capacity, and the server flooded with 500s — ironically *because* Redis was too fast.

**The bug I'm most proud of finding**

The 500 errors were embarrassing, but they weren't the real problem. The real problem was what the crash *caused silently*.

Here's the database going into the test — 10,000,000 units of stock:

<img width="951" height="216" alt="Database before V3 initial test: 10,000,000 stock" src="https://github.com/user-attachments/assets/53f5ee52-e519-4231-9002-5c8b7702d57f" />

After the test finished, I checked two things. The Redis stock counter read `9,967,138` — meaning Redis believed **32,862 units** had been sold. Then I ran:

```sql
SELECT count(*) FROM "Order";
```

Postgres had only recorded **26,261 orders**. Redis said 32,862 sales. Postgres only knew about 26,261 of them.

**6,601 units were sold, decremented, and permanently unrecorded.**

Here's exactly what happened: the Lua script decremented Redis first — that always succeeded instantly. Then the handler attempted the Postgres write. When the connection pool was exhausted and Prisma threw `P2028`, that write failed and returned a 500 to the client. But the Redis decrement had *already happened* with no rollback and no retry queued. The stock was gone from Redis. The order record was never created. From the customer's perspective their purchase failed. From the inventory's perspective, the unit was sold.

| | Units sold (Redis) | Orders recorded (Postgres) |
|---|---|---|
| Expected | 32,862 | 32,862 |
| Actual | 32,862 | 26,261 |
| **Divergence** | — | **6,601 permanently lost** |

The error rate graph shows exactly when Postgres started buckling — each spike is a batch of writes that failed after Redis had already committed the corresponding decrements:

<img width="1858" height="901" alt="Grafana: 500 error rate during V3 initial — each spike is Postgres writes failing after Redis already decremented stock, with no compensation" src="https://github.com/user-attachments/assets/65fe4db4-d6c5-46c6-897d-d98677d01595" />

While those writes were failing, requests were blocking on a DB connection that would never arrive — which shows up as p95 latency climbing in lockstep with the error spikes:

<img width="1864" height="910" alt="Grafana: p95 latency during V3 initial — rises alongside 500 errors as threads pile up waiting on Postgres connections that are exhausted" src="https://github.com/user-attachments/assets/dd121b00-bcc6-421a-901c-319b9b4ed360" />

After the test, the database tells the whole story. Redis at `9,967,138`, Postgres with records for only `26,261` orders — 6,601 units decremented in Redis with no trace anywhere in the order table:

<img width="950" height="220" alt="Database after V3 initial: Redis stock = 9,967,138 but Postgres only has 26,261 orders — 6,601 units consumed but never recorded" src="https://github.com/user-attachments/assets/16a1ac27-9d80-487f-8804-4877130d0d0f" />

The full k6 summary shows the error rate that produced this divergence. These weren't just failed requests — each 500 here is an inventory unit that was consumed and silently discarded:

<img width="1892" height="1142" alt="k6 test results for V3 initial: significant error rate under 1000 VU load with abundant stock — each error is a permanently lost order record" src="https://github.com/user-attachments/assets/35cf652d-4a5b-447e-9bb8-7e1110e844f8" />

This is what made V3 Initial dangerous rather than just broken. It passed my initial tests because scarce stock masked the bottleneck. It only failed when the conditions changed. A system that silently loses data under specific load conditions is far harder to catch than one that crashes outright — and far more damaging if it ever reaches production.

The root cause was architectural: I coupled a fast, non-blocking operation (Redis decrement) directly to a slow, failure-prone one (Postgres write) in the same synchronous request cycle. If the slow step failed, there was no way back. That's what needed to change.

**What I learned:** A fast system that loses data silently is worse than a slow system that doesn't. And a system that only works under the right conditions isn't a working system — it's a bug waiting for the conditions to change.

---

## V4: Redis + BullMQ Message Queue

<img width="2000" height="614" alt="V4 Architecture Diagram — Redis atomic decrement via Lua, BullMQ async buffer, worker-to-Postgres pipeline with DLQ rollback" src="https://github.com/user-attachments/assets/ecc2a972-b884-4ccb-928c-2c23c53f0e63" />

**What I built:** V3 proved that coupling a fast Redis decrement directly to a slow, synchronous Postgres write inside the same request was a recipe for ghost reservations. The Lua script was never the problem — it was what came *after* it. So I kept the identical Lua decrement logic and removed the synchronous database call entirely from the request path.

Instead, I introduced BullMQ as an asynchronous buffer between Redis and Postgres. The service (`buy.queue.service.js`) runs the same Lua script against Redis, and when the result is `>= 0` (stock available), it pushes a lightweight job onto a BullMQ `orders` queue via `addJob(userId, productId, stock)` and immediately returns a `202 Accepted`. The entire request lifecycle ends there — no Postgres interaction on the hot path at all.

The cache miss path (`-1`) is unchanged from V3: the service falls back to Postgres to fetch the current stock, seeds it into Redis with `NX` and a TTL of 3600 seconds, and recursively retries the buy call. The out-of-stock path (`-2`) returns a `400` rejection instantly.

The queue is configured with 3 retry attempts using exponential backoff starting at 1 second. The job payload carries the `userId`, `productId`, and current `stock` value.

**The worker process:** A separate Node.js process (`placeOrderWorker.js`) consumes jobs from the `orders` queue with a concurrency of 10. Each job runs a `prisma.$transaction` that does two things atomically:
1. Decrements the product stock by 1 in Postgres
2. Creates an order record with status `CONFIRMED`

If the transaction fails, BullMQ retries the job. If the job exhausts all 3 attempts, the worker checks `job.attemptsMade === (maxAttempts - 1)` and on the final failure, pushes a new job to a `failed_orders` queue — my Dead Letter Queue. The `FAILED` metric is incremented at this point.

### How I Made Workers Visible to Grafana

One of the biggest lessons I learned building V4 is that **workers are invisible by default**. They run in an entirely separate Node.js process — Prometheus has no idea they exist unless you explicitly expose metrics from them.

I solved this by spinning up a lightweight HTTP server *inside each worker process*. The `placeOrderWorker.js` imports the same `metricsService` singleton and starts an `http.createServer` on a port defined by `WORKER_METRICS_PORT`. The `dlqWorker.js` does the same on `DLQWORKER_METRICS_PORT`. Each server exposes a `/metrics` endpoint that Prometheus scrapes independently.

This means Prometheus now has three scrape targets:
- The Express API server (port 3000)
- The Place Order Worker (`WORKER_METRICS_PORT`)
- The DLQ Rollback Worker (`DLQWORKER_METRICS_PORT`)

The `yoink_total_orders` counter tracks every stage of an order's lifecycle through its `status` label:
- `ACCEPTED` — Lua script succeeded, job was added to the queue (API server)
- `REJECTED` — Lua script returned `-2`, out of stock (API server)
- `CONFIRMED` — Worker successfully wrote the order to Postgres (Place Order Worker)
- `FAILED` — Worker exhausted all retries, job sent to DLQ (Place Order Worker)
- `ROLLBACK_SUCCESSFUL` — DLQ Worker incremented Redis stock back (DLQ Worker)
- `ROLLBACK_FAILED` — DLQ Worker itself failed to rollback (DLQ Worker)

This gives me a full end-to-end pipeline view in Grafana — from the moment a request hits Express to the moment the order is confirmed in Postgres, or rolled back in Redis.

<img width="1904" height="1068" alt="Prometheus Targets page — all three scrape targets (API server, Place Order Worker, DLQ Worker) showing as UP" src="https://github.com/user-attachments/assets/18f280da-b8ec-4e4f-a097-40ba9a474048" />

### The Dead Letter Queue — Self-Healing Inventory

The DLQ is the critical piece that makes V4 production-resilient and directly solves the ghost reservation problem from V3.

Here's the failure scenario: the Place Order Worker pulls a job, but the `prisma.$transaction` fails — network timeout, disk full, Postgres is down. BullMQ retries the job 3 times with exponential backoff. If the job fails all 3 attempts, the worker pushes a new job to the `failed_orders` queue with the `productId` and a reason of `'Max retries reached'`.

A dedicated DLQ Rollback Worker (`dlqWorker.js`) watches the `failed_orders` queue. When it picks up a dead job, it runs a single operation: `redisClient.incr(\`product:${productId}\`)` — incrementing the stock back by 1 in Redis. The `ROLLBACK_SUCCESSFUL` metric is incremented, and the unit is available for purchase again.

This completely eliminates the data divergence I discovered in V3. The system is eventually consistent by design:
- If the DB write succeeds → order is `CONFIRMED`, stock is synced across both stores.
- If the DB write permanently fails → stock is returned to Redis, as if the purchase never happened. No ghost reservation. No lost inventory.

---

### Test 1: Spike Load — 5,000 Items, 1,000 Concurrent Users

V3 handled this scarce stock scenario without errors — the Lua script ran out of stock quickly, and most requests hit the `-2` out-of-stock path before ever touching Postgres. But V3 was also making synchronous Postgres writes for every successful decrement. With only 5,000 units, the total number of DB writes was small enough that the connection pool held up. This test establishes V4's baseline behavior under the same conditions, before pushing it to the scale that broke V3.

Here's the database going in — 5,000 units of stock:

<img width="902" height="219" alt="Database before V4 Test 1: product has 5,000 stock" src="https://github.com/user-attachments/assets/57308053-6983-4120-acf9-db8ce1834209" />

The test ran, and the Grafana dashboard tells the full story. This is the total orders panel — look at how `ACCEPTED` and `CONFIRMED` climb in lockstep, meaning the worker is draining the queue nearly as fast as the API is filling it. Once stock runs out, `REJECTED` climbs as the remaining requests are correctly turned away:

<img width="1882" height="932" alt="Grafana: total orders processed — ACCEPTED and CONFIRMED climb together, REJECTED rises after stock runs out" src="https://github.com/user-attachments/assets/c5eb9029-99ce-4928-961f-6d9b55fc2026" />

Unlike V3, there are zero `P2028` errors and zero 500s. The only errors are `400` out-of-stock rejections — exactly the behavior we want:

<img width="1882" height="934" alt="Grafana: error rate showing only 400 out-of-stock errors, no 500s, no connection pool exhaustion" src="https://github.com/user-attachments/assets/ce80f413-7b5f-46ea-888d-20bf5d11e5ff" />

The p95 latency tells the clearest story. Because the API never touches Postgres — it just decrements Redis and pushes to BullMQ — the peak latency was only **8.6ms**. Compare this to V2's **476ms** during peak contention:

<img width="1881" height="929" alt="Grafana: p95 latency for V4 — peak at 8.6ms, orders of magnitude lower than V2" src="https://github.com/user-attachments/assets/8137816f-55fe-4384-ae56-de07f2302832" />

The throughput held steady. The test script fired at 1,000 req/s peak for 30 seconds — Prometheus computes the rate as an average over its scrape interval, so the graph displays ~580 req/s rather than the instantaneous 1,000 req/s peak. The server handled the full burst without breaking a sweat:

<img width="1881" height="935" alt="Grafana: API requests per second — Prometheus averages to ~580 req/s over scrape interval, actual peak was 1,000 req/s" src="https://github.com/user-attachments/assets/5b400388-b7b5-4437-9c17-7d7d42622445" />

The full k6 summary confirms the results. The errors shown are entirely `400` out-of-stock rejections — not a single 500:

<img width="1894" height="1135" alt="k6 test results for V4 Test 1: all errors are 400 out-of-stock, zero 500s, zero connection pool failures" src="https://github.com/user-attachments/assets/3d16e486-d66a-40a9-92e3-ff2efe84d963" />

**Inventory Integrity Check:**

After the test completed and the worker fully drained the queue, I checked both data stores. The database shows stock at 0 with exactly 5,000 orders:

<img width="900" height="216" alt="Database after V4 Test 1: stock = 0, order count = 5,000 — perfect integrity" src="https://github.com/user-attachments/assets/37514ea4-4981-4468-b646-a26debb5ffa1" />

Redis Commander confirms the same — `product:` key at `0`:

<img width="992" height="603" alt="Redis Commander: stock key showing 0 — perfectly in sync with Postgres" src="https://github.com/user-attachments/assets/0a11ab97-3761-44dc-963e-5fe78438590e" />

Redis stock: `0`. Postgres stock: `0`. Postgres orders: `5,000`. Perfect sync — no divergence, no ghost reservations.

---

### Test 2: Scale Stress — 10,000,000 Items

This is the test that specifically broke V3. With 10,000,000 units of stock, almost every request during the spike succeeds at the Lua script level — meaning in V3, every single one tried to write to Postgres synchronously. That's what caused the `P2028` flood and 6,601 ghost reservations.

Here's the database going in — 10,000,000 units:

<img width="902" height="219" alt="Database before V4 Test 2: product has 10,000,000 stock" src="https://github.com/user-attachments/assets/904a1793-795c-4de4-84de-74657c434737" />

With V4, it doesn't matter how much stock there is. The API pushes jobs to BullMQ at full speed, and the worker drains them at its own controlled pace. The queue absorbs the burst. This is the total orders panel — `ACCEPTED` jumps up during the spike, and `CONFIRMED` climbs steadily as the worker processes the backlog at a sustainable rate:

<img width="1880" height="926" alt="Grafana: total orders — ACCEPTED spikes during burst, CONFIRMED climbs steadily as worker drains the queue" src="https://github.com/user-attachments/assets/89b60c60-85ab-4745-85d8-c914260d5bd6" />

Even with the massive inventory, p95 latency peaked at only **26.2ms**. In V3, this same test caused latency to spiral as threads piled up waiting on Postgres connections that were exhausted:

<img width="1858" height="907" alt="Grafana: p95 latency for V4 10M test — peak at 26.2ms, no latency spiraling" src="https://github.com/user-attachments/assets/80553e00-5404-4248-9466-41f7a4fba37e" />

The k6 test script fired at a peak of 2,000 req/s, but the Grafana panel shows approximately 1.3–1.32k req/s. This is expected — Prometheus uses the `rate()` function which calculates the per-second average over an interval (typically 15 seconds), smoothing out the instantaneous peak. The actual burst of 2,000 req/s gets averaged down, but the important thing is the graph shows clean, stable throughput with no dips or errors:

<img width="1851" height="904" alt="Grafana: API requests per second during 10M test — Prometheus rate() averages to ~1–1.3k req/s over scrape interval, actual k6 peak was 2,000 req/s" src="https://github.com/user-attachments/assets/485429f6-571a-4bc8-b2f5-4e1d2b1e5ea0" />

The k6 summary and web dashboard confirm zero errors and stable performance throughout:

<img width="1894" height="1137" alt="k6 terminal results for V4 10M test: zero errors, stable response times at 2k req/s peak" src="https://github.com/user-attachments/assets/3e6082a8-129b-47e8-920d-818d4947b9e5" />

<img width="1878" height="1072" alt="k6 web dashboard for V4 10M test: clean request distribution and response time histograms" src="https://github.com/user-attachments/assets/2ea5c5f4-dd1d-424b-b4d1-0a7e66bd351a" />

**Final Inventory Verification:**

After the worker finished draining the queue, both data stores are in sync:

<img width="903" height="219" alt="Database after V4 Test 2: stock decremented correctly, orders match" src="https://github.com/user-attachments/assets/c9a49c2d-6496-48f3-90a6-6d0a319aa534" />

<img width="948" height="598" alt="Redis Commander after V4 Test 2: stock key matches Postgres — no divergence" src="https://github.com/user-attachments/assets/28c06731-85f7-4d11-b922-fa3c35de463a" />

The test that catastrophically broke V3 — producing 6,601 ghost reservations — runs cleanly on V4 with zero data loss.

---

### Test 3: DLQ Resilience — Intentional Database Failure

This is the test that proves V4 isn't just fast — it's **self-healing**. I deliberately killed the Postgres container mid-test to simulate a real database outage, then observed the full pipeline: API → BullMQ → Worker failure → DLQ → Rollback.

**Steps:**
1. Seeded 6,000 units of stock. Started the k6 spike test at 1,000 req/s peak.
2. Midway through the test, while the Place Order Worker was actively draining the queue, I ran:
   ```bash
   docker stop yoink-postgres
   ```
3. The worker immediately started failing — every `prisma.$transaction` threw a connection error. BullMQ retried each job 3 times with exponential backoff.
4. After all retries were exhausted, the worker pushed each permanently failed job to the `failed_orders` DLQ queue and incremented the `FAILED` metric.
5. The DLQ Rollback Worker picked up each dead job and ran `redisClient.incr(\`product:${productId}\`)`, returning the stock to Redis and incrementing `ROLLBACK_SUCCESSFUL`.

Here's the database going in — 6,000 units of stock:

<img width="903" height="220" alt="Database before V4 Test 3: product has 6,000 stock" src="https://github.com/user-attachments/assets/0b76f916-52d0-42f1-b93c-43ffb923f30f" />

The Grafana total orders panel shows the entire failure and recovery story in a single graph:

<img width="1852" height="904" alt="Grafana: total orders during DLQ test — showing ACCEPTED, CONFIRMED, FAILED, and ROLLBACK_SUCCESSFUL counters throughout the database outage" src="https://github.com/user-attachments/assets/cb5e7598-7b70-4769-a803-e4c83b9bd93a" />

There's something interesting in this graph: the `ACCEPTED` count exceeds 6,000 — the initial stock. This is the DLQ doing exactly what it was designed to do. Here's the timeline of what happened:

1. Before the crash, orders flowed normally: the Lua script decremented stock, the service enqueued a job (`ACCEPTED`), and the worker wrote to Postgres (`CONFIRMED`).
2. When I stopped Postgres, jobs in the queue started failing. After 3 retries, the worker sent them to DLQ (`FAILED`).
3. The DLQ Rollback Worker picked up those dead jobs and incremented the stock *back* into Redis (`ROLLBACK_SUCCESSFUL`). Those units were now available for purchase again.
4. Meanwhile, the k6 test was still running and sending requests. New requests hit the Lua script, found the rolled-back stock available, decremented it, and generated new `ACCEPTED` events.
5. Those new jobs also failed at the worker (Postgres was still down), got DLQ'd, rolled back, and the cycle continued until the test ended. Crucially, the `ROLLBACK_SUCCESSFUL` and `FAILED` counts match exactly — every item that failed to persist in Postgres was returned to Redis without exception. No unit was permanently lost; every failed order was fully compensated.

Each unit that failed went through this cycle: **ACCEPTED → FAILED → ROLLBACK → back in Redis → re-ACCEPTED**. The `ACCEPTED` counter kept incrementing with each cycle, but no stock was ever lost — the DLQ kept returning it to Redis. Once the test stopped sending requests and the queue fully drained, the system settled to its final consistent state: **confirmed Postgres orders + remaining Redis stock = 6,000**.

The p95 latency stayed low at a peak of **18.1ms** throughout, because even during the database outage, the API layer was completely unaffected — it only talks to Redis and BullMQ:

<img width="1856" height="901" alt="Grafana: p95 latency during DLQ test — 18.1ms peak, API completely unaffected by Postgres outage" src="https://github.com/user-attachments/assets/69f936a8-0566-4102-86bb-4dc8ad3b2132" />

API throughput remained clean. The server kept accepting and rejecting requests at full speed, oblivious to the Postgres failure happening downstream:

<img width="1847" height="900" alt="Grafana: API requests per second during DLQ test — stable throughput throughout the database outage" src="https://github.com/user-attachments/assets/bf3a5285-deaa-4b06-9208-58295b7d90a7" />

The terminal logs tell the behind-the-scenes story. You can see the Place Order Worker logging `"Job exhausted all retries. Sending to DLQ."` followed by the DLQ Rollback Worker logging `"Rollback successful for product: ..."` as it returns each unit back to Redis:

<img width="1898" height="1140" alt="Terminal: Place Order Worker exhausting retries and sending to DLQ, DLQ Worker logging successful rollbacks" src="https://github.com/user-attachments/assets/68d6a116-0a1f-4a51-a3d5-e5a2be4a2c14" />

**Final State After Outage:**

After the DLQ worker finished processing all failed jobs, I checked the final state. The database shows the orders that were confirmed *before* Postgres went down:

<img width="895" height="214" alt="Database after V4 Test 3: confirmed orders recorded before the crash, stock reflects only those" src="https://github.com/user-attachments/assets/9400dfb4-2cde-4887-a3af-f5ac0ab3fdc1" />

Redis Commander shows the remaining stock — the units that the DLQ worker returned after the crash:

<img width="950" height="588" alt="Redis Commander after V4 Test 3: stock restored by DLQ rollbacks, perfectly accounting for unconfirmed orders" src="https://github.com/user-attachments/assets/5196c454-d368-4bc9-be00-4e09976e4aa3" />

The k6 summary confirms the errors were all `400` out-of-stock rejections — not 500s from a crashing server. The API layer never knew Postgres went down:

<img width="1877" height="1134" alt="k6 test results for V4 Test 3: errors are 400 out-of-stock rejections, not 500 server failures" src="https://github.com/user-attachments/assets/756eb9b6-b143-4fd2-abb3-67166a50cfbc" />

The confirmed Postgres orders and the remaining Redis stock add up to exactly **6,000** — every unit is accounted for. Some were sold and written to Postgres before the crash. The rest were returned to Redis by the DLQ worker after the crash. No units were lost. No ghost reservations. The system healed itself without any manual intervention.

In V3, this exact scenario would have produced thousands of permanently lost inventory units — decremented in Redis, never recorded in Postgres, never rolled back. V4's DLQ makes that architecturally impossible.

---

## What I'd Do Differently If I Started Over

If I started this project over, I wouldn't assume that more database connections equal more throughput. Setting a Postgres pool size to 1,000 was a massive mistake that just thrashed the OS scheduler and consumed RAM.

More importantly, I would never write a line of code or install a shiny new library without measuring the system first. I learned more by running k6 benchmarks and explicitly analyzing the observability data in Grafana than I did writing any actual code. Every version in this case study wasn't born from a guess — it was born from a metric that pointed at a specific bottleneck. In the future, I will let the metrics dictate the architecture, rather than guessing what the bottleneck might be.
