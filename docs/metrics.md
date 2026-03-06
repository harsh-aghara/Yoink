# Observability & Metrics: The Distributed Monitoring Deep-Dive

In high-concurrency systems like **Yoink!**, monitoring isn't just about knowing if the server is "up." It's about detecting silent data corruption, measuring system-wide latency, and ensuring eventual consistency across distributed processes.

This document details the complete monitoring architecture, from the core metrics to the multi-process scraping strategy.

---

## 1. Core Metrics & Definitions

All metrics are centralized in `src/services/metrics.service.js` using the `prom-client` library.

### `yoink_http_request_duration_seconds` (Histogram)
*   **Purpose:** Measures the latency of every HTTP request.
*   **Labels:** `method`, `route`, `status_code`.
*   **Buckets:** `[0.005, 0.01, 0.025, 0.05, 0.1, 0.5, 1, 2.5]`
*   **Senior Insight:** We use specific buckets to capture the "Goldilocks Zone" of performance. A spike in the `1s+` bucket immediately signals database contention (V2) or network lag.

### `yoink_inventory_stock_level` (Gauge)
*   **Purpose:** Tracks the real-time inventory count.
*   **Labels:** `product_id`.
*   **Strategy:** This is a "snapshot" metric. In V3, comparing this gauge in Redis vs. the actual row count in Postgres allowed us to mathematically prove the "Inventory Gap."

### `yoink_total_orders` (Counter)
*   **Purpose:** Tracks the entire lifecycle of an order.
*   **Labels:** `status`.
*   **Logic:** This is our most critical metric. The `status` label evolves across the four architectural versions (see below).

---

## 2. Metrics Evolution: V1 to V4

As the architecture grew more complex, our metrics had to become more granular to maintain visibility.

### Phase 1 & 2: Synchronous CRUD (V1/V2)
*   **`status="CONFIRMED"`**: Incremented when the DB transaction succeeds.
*   **`status="FAILED"`**: Incremented on any `5xx` error or connection timeout.
*   **Observation:** In V1, we saw `CONFIRMED` counts far exceed the initial `inventoryGauge`, proving the race condition.

### Phase 3: Redis Sync (V3)
*   **The "Inventory Gap" Detection:** We introduced logic to track successful Redis decrements vs. successful Postgres writes.
*   **Data Divergence:** When `Redis Success Count > Postgres Success Count`, it signaled that units were being sold in the cache but lost in the database.

### Phase 4: Event-Driven Resilience (V4)
This is the current production standard, using a multi-stage lifecycle:
1.  **`status="ACCEPTED"`**: API received the request, Redis decremented stock, and the job was enqueued.
2.  **`status="REJECTED"`**: Immediate rejection at the entry point, most commonly due to **Out of Stock** detected by the Lua script.
3.  **`status="CONFIRMED"`**: The background worker (`placeOrderWorker.js`) successfully wrote the order to Postgres.
4.  **`status="FAILED"`**: The DB write failed even after 3 retries (Job moved to DLQ).
5.  **`status="ROLLBACK_SUCCESSFUL"`**: The DLQ worker (`dlqWorker.js`) successfully incremented the Redis stock back, restoring consistency.
6.  **`status="ROLLBACK_FAILED"`**: A critical error where even the recovery attempt failed.

---

## 📡 3. Distributed Scraping Infrastructure

Because our workers run in separate processes, they cannot share memory. We implemented a **Multi-Endpoint Scraping** pattern.

| Endpoint | Port | Source | Description |
| :--- | :--- | :--- | :--- |
| `/metrics` | `3000` | API Server | Global HTTP metrics and `ACCEPTED` counts. |
| `/metrics` | `3003` | Order Worker | `CONFIRMED` and `FAILED` counts for background writes. |
| `/metrics` | `3004` | DLQ Worker | `ROLLBACK` success/failure counts. |

### Technical implementation:
In our background workers, we use the native Node.js `http` module to expose metrics without the overhead of Express:
```javascript
http.createServer(async (req, res) => {
    if (req.url === '/metrics') {
        res.setHeader('Content-Type', metricsService.getMetricsContentType());
        res.end(await metricsService.getMetrics());
    }
}).listen(PORT, '0.0.0.0');
```

---

##  4. The Prometheus-Docker Bridge

Prometheus runs in a Docker container, but our workers run on the host. We bridged this gap using `host.docker.internal`.

**Prometheus Configuration (`prometheus.yml`):**
```yaml
scrape_configs:
  - job_name: 'yoink_worker'
    static_configs:
      - targets: ['host.docker.internal:3003']
```
This configuration allows Prometheus to reach out of its container to scrape our high-performance workers.

---

##  5. Why this Impresses Senior Engineers

1.  **Observability-Driven Development (ODD):** We didn't just add metrics at the end; we used metrics to **discover** the bugs in V1 and V3.
2.  **Granular Process Insights:** By scraping each worker individually, we can monitor the **Event Loop Lag** and **Memory Usage** of the background processes separate from the API.
3.  **Eventual Consistency Proof:** The `yoink_total_orders` counter with its evolving labels proves that we have a "closed-loop" system where every failure is eventually accounted for and rolled back.
4.  **Middleware Precision:** Our `trackLatency` middleware ensures that even if a request fails, we still capture its original route and status code, preventing "Ghost Errors" in our Grafana dashboards.
