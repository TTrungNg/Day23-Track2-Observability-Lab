# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Việt Trung
**Lab repo URL:** https://github.com/TTrungNg/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
{
  "docker": {
    "ok": true,
    "version": "28.3.2"
  },
  "compose_v2": {
    "ok": true,
    "version": "2.38.2-desktop.1"
  },
  "ram_gb_available": 7.65,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

Machine: Apple M-series (ARM64/darwin), Docker Desktop 28.3.2, 7.65 GB RAM available. All 9 required ports were free before startup.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

The AI Service Overview dashboard shows: Request Rate (req/s), P99 Latency (seconds), Active Inference Gauge (in-flight requests), Token Throughput (input + output tokens/s), Quality Score (eval-as-metric), and GPU Utilization. After running `make load` with 10 concurrent users for 60s (~1011 requests total, 0 failures), all panels populated with real data. P99 latency stabilised around 260ms on M-series hardware.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

The SLO Burn Rate dashboard uses the `slo-burn-rate.yml` multi-window rules. The fast-burn window (1h/5m) and slow-burn window (6h/30m) both showed 0 burn rate during the healthy load test — as expected since the mock inference service had 0 errors and sub-second latency well within the 2s SLO budget.

### Alert fire + resolve

| When                 | What                                 | Evidence                             |
| -------------------- | ------------------------------------ | ------------------------------------ |
| 2026-05-12T04:02:32Z | killed `day23-app` via `docker stop` | —                                    |
| 2026-05-12T04:03:52Z | `ServiceDown` fired in Alertmanager  | screenshot `alertmanager-firing.png` |
| 2026-05-12T04:06:25Z | restarted app via `docker start`     | —                                    |
| 2026-05-12T04:06:33Z | alert resolved                       | screenshot `slack-resolved.png`      |

Alert fired ~80 seconds after the container was stopped (consistent with `for: 1m` + one 15s scrape cycle). Alertmanager routed to `#oncall` (critical severity) and `#observability` with `send_resolved: true`.

### One thing surprised me about Prometheus / Grafana

The `{{ env "VAR" }}` Go-template syntax works inside Alertmanager _notification templates_ (title/text fields) but NOT in config-level URL fields like `api_url`. Alertmanager evaluates config YAML at parse time without template expansion — the URL field must be a literal string or resolved via `slack_api_url_file`. Switching to write the webhook URL to `/tmp/slack_url.txt` via a shell entrypoint and referencing it with `slack_api_url_file` fixed the issue.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

Trace `c515d9b3cfd7bd21a70bfea555a46f3f` captured in Jaeger for `POST /predict`. The parent span `predict` has 3 child spans in sequence: `embed-text` (5ms), `vector-search` (10ms), `generate-tokens` (~165ms). The GenAI semantic convention attributes are set on the `generate-tokens` span: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason=stop`.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{
  "model": "llama3-mock",
  "input_tokens": 4,
  "output_tokens": 54,
  "quality": 0.82,
  "duration_seconds": 0.1597,
  "trace_id": "c515d9b3cfd7bd21a70bfea555a46f3f",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-05-12T03:49:59.431781Z"
}
```

The `trace_id` field (`c515d9b3cfd7bd21a70bfea555a46f3f`) links directly to the Jaeger trace. This enables log→trace correlation: paste the trace_id into Jaeger search to jump to the exact distributed trace for any log event.

### Tail-sampling math

At peak load (~18 req/s during `make load`), the OTel Collector tail-sampling policy retains:

- **All ERROR traces** (keep-errors policy) — forced failures via `?fail=true` would be 100% retained
- **All traces with latency > 2000ms** (keep-slow policy) — none during normal operation
- **1% of remaining healthy traces** (probabilistic-1pct policy)

With 18 req/s healthy traffic, expected retention = 18 × 0.01 = **0.18 traces/s**, or ~11 traces/minute. During the 60s load test, approximately 10–12 healthy traces were stored in Jaeger out of ~1011 requests. The `decision_wait: 30s` window means spans are buffered for 30s before the sampling decision is made.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

`prompt_length` shifted from N(50,15) to N(85,20) — a 70% mean shift. `response_quality` shifted from Beta(8,2) (high quality, mean≈0.8) to Beta(2,6) (low quality, mean≈0.25) — a severe quality regression. Both trigger PSI >> 0.2 (the "significant drift" threshold). `embedding_norm` and `response_length` were kept identical between reference and current datasets, correctly showing PSI < 0.02.

### Which test fits which feature?

| Feature            | Best test in production | Reason                                                                                                                                                                                                                                                                                                                                                           |
| ------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt_length`    | **PSI**                 | Continuous, approximately normal. PSI gives a single scalar per monitoring window — easy to alert on with a static threshold (> 0.2 = moderate, > 0.25 = alert). Stable under sample size variation common in production.                                                                                                                                        |
| `embedding_norm`   | **KS test**             | Near-constant distribution (very low variance). KS is distribution-free and sensitive to shape changes. PSI would require choosing bin edges carefully; KS avoids that.                                                                                                                                                                                          |
| `response_length`  | **PSI**                 | Right-skewed continuous distribution with known production range. PSI handles the tail well via custom binning and is interpretable as "% shift in population". KL is asymmetric which is less convenient for monitoring.                                                                                                                                        |
| `response_quality` | **KS + PSI together**   | Beta-distributed [0,1] with dramatically different shape (bimodal risk). KS detects any distributional shift including shape changes; PSI quantifies magnitude. Using both gives both detection sensitivity and severity scoring. For a production quality metric, MMD (Maximum Mean Discrepancy) would also be appropriate as it captures higher-order moments. |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day source to integrate would be the **Day 19 vector store (Qdrant)**. Unlike the Day 20 llama.cpp serving layer which natively exposes a Prometheus `/metrics` endpoint, Qdrant's metrics are accessible via its REST API at `/metrics` (Prometheus format) only if the Qdrant service is actually running and the `DAY19_QDRANT_URL` env var is configured. The integration stub in `monitor-day19-vector-store.py` must bridge between Qdrant's collection-level stats (collection size, vector count, search latency) and the Prometheus data model — requiring a custom scraper that runs as a sidecar. This is harder than cloud infra metrics (Day 16) which are pulled via well-documented APIs with official exporters.

---

## 6. The single change that mattered most

The single most impactful change in this stack was **adding `trace_id` as a field in every structured log line** (see `instrumentation.py:bind_log` + `main.py:log.info(..., trace_id=trace_id)`). Without it, metrics, traces, and logs are three separate silos: you can see _that_ latency spiked in Grafana, see _what_ slow spans were in Jaeger, and see _some_ errors in logs — but you cannot connect a specific slow request to its exact log output. Adding `trace_id` to logs collapses the gap between the "metrics → alert" path and the "trace → root cause" path into a single lookup.

This connects directly to the **Three Pillars of Observability** from the deck (§2): metrics give you _when and how much_, traces give you _where in the call graph_, and logs give you _why_. The trace_id is the foreign key that joins all three. In practice, when a P99 latency alert fires, the SRE can: (1) open the Grafana alert → see the spike window, (2) query Jaeger for slow traces in that window, (3) copy the trace_id → search structured logs → read the exact error message or model output. This workflow — which only works because the trace_id propagates into the log — cuts mean-time-to-root-cause from "minutes of log grepping" to "three clicks". The change was one line of code but enabled the entire observability value chain.
