# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _your name_
**Submission date:** _YYYY-MM-DD_
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (27.3.0)
Compose v2:    OK  (2.32.4)
RAM available: 16.0 GB (OK)
Ports free:    OK
Report written: 00-setup/setup-report.json
```

**setup-report.json:**
```json
{
  "docker": {
    "ok": true,
    "version": "27.3.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "2.32.4"
  },
  "ram_gb_available": 16.0,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The thing that surprised me most was how the **multi-window multi-burn-rate alerting** (deck §6) actually works in practice with Prometheus recording rules. Before this lab, I assumed SLO alerting was a simple "error rate > threshold" check. But the dual-window approach — requiring both a short window (5m) AND a long window (1h) to simultaneously exceed 14.4× burn rate — eliminates the false positives that a single window would produce. What really clicked was realizing that a transient 10-second blip of 100% errors would trigger a single-window alert but would *not* trip the multi-window condition because the 1h window averages it out. The recording rules (`inference:fail_ratio:rate5m`, `rate30m`, `rate1h`, `rate6h`) as pre-computed PromQL expressions made this elegant enough that I now understand why Google SRE calls this "the right way to alert on SLOs."

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"event": "prediction served", "level": "info", "timestamp": "2026-06-29T12:34:56.789Z",
 "model": "llama3-mock", "input_tokens": 3, "output_tokens": 42, "quality": 0.833,
 "duration_seconds": 0.2345,
 "trace_id": "abcdef1234567890abcdef1234567890",
 "logger": "main"}
```

The `trace_id` field in the structlog JSON line matches the trace ID visible in Jaeger, and Grafana's Loki datasource has a derived field configured with regex `"trace_id":"([a-fA-F0-9]+)"` that auto-links from log line → Jaeger trace.

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

**Setup:**
- Policy 1: Keep 100% of error traces
- Policy 2: Keep 100% of slow traces (latency > 2s)
- Policy 3: Keep 1% of healthy traces (probabilistic)

**Assume typical traffic distribution (1% errors, 1% slow, 98% healthy):**

```
sampled = N × (P(error) × 1.0 + P(slow ∧ ¬error) × 1.0 + P(healthy) × 0.01)
        = N × (0.01 × 1.0 + 0.01 × 1.0 + 0.98 × 0.01)
        = N × (0.01 + 0.01 + 0.0098)
        = N × 0.0298
        ≈ 3% retention
```

**Cost reduction:** 1 - 0.0298 = ~97% reduction vs. retain-everything.

**Buffer cost:** The collector holds 30s of traces in memory (decision_wait: 30s). With 50,000 trace slots and ~10 spans/trace, memory usage ≈ 50 MB. This handles up to ~1,500 traces/sec before buffer overflow.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 0.5231,
    "kl": 0.2845,
    "ks_stat": 0.4100,
    "ks_pvalue": 0.000001,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0432,
    "kl": 0.0214,
    "ks_stat": 0.0580,
    "ks_pvalue": 0.342100,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0517,
    "kl": 0.0258,
    "ks_stat": 0.0720,
    "ks_pvalue": 0.185300,
    "drift": "no"
  },
  "response_quality": {
    "psi": 0.8913,
    "kl": 0.4169,
    "ks_stat": 0.5350,
    "ks_pvalue": 0.000000,
    "drift": "yes"
  }
}
```

**Analysis:** Two features drift significantly — `prompt_length` (PSI=0.52, shifted from N(50,15) to N(85,20)) and `response_quality` (PSI=0.89, shifted from Beta(8,2) to Beta(2,6)). Two features remain stable: `embedding_norm` and `response_length`.

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

| Feature | Best test | Reason |
|---|---|---|
| **prompt_length** | **KS (Kolmogorov-Smirnov)** | Continuous, unbounded, Gaussian-like. KS detects both shift in mean AND change in variance. It's non-parametric and makes no distributional assumptions. PSI would also work but is less sensitive to subtle shifts because it discretizes into bins. |
| **embedding_norm** | **MMD (Maximum Mean Discrepancy)** | Embedding norms live in high-dimensional space derived from a model — MMD with a kernel (RBF) captures manifold structure that discretization-based tests (PSI, KL) or ECDF-based tests (KS) would miss. In practice, PSI works fine here because we're looking at the norm (1-D), but for the raw embedding vectors, MMD is the right choice. |
| **response_length** | **PSI (Population Stability Index)** | Response length is bounded and interpretable in bins (e.g., 0-50, 51-100, etc.). PSI is the financial industry standard for exactly this kind of bounded continuous variable. The binning also naturally smoothes out noise from the log-normal tail of response lengths. |
| **response_quality** | **KL divergence** | Quality scores are bounded [0,1] and skewed (Beta distribution). KL(P_ref || P_cur) is more sensitive than PSI to changes in the tail of a bounded distribution — critical for quality metrics where a small shift in the low-quality tail has outsized business impact. KL's asymmetry also tells us *direction*: did scores get worse or better? |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest metric to expose would be **Day 20's llama.cpp tokens-per-second** (`day20_llamacpp_tokens_per_second`). Unlike Day 19's Qdrant (which natively exposes Prometheus metrics at `/metrics`), llama.cpp's HTTP server has **no built-in Prometheus endpoint** — it would require either patching the C++ codebase or running a sidecar exporter that scrapes llama.cpp's internal stats API and re-exposes them as Prometheus metrics. The lab provides a stub script (`monitor-day20-llama-cpp.py`) that emits fake-shaped metrics, but in production you'd need to either (a) modify llama.cpp's source to emit counters, (b) run a Prometheus exporter sidecar like `llama-cpp-prometheus-exporter`, or (c) parse llama.cpp's stdout logs with a log-shipper like Alloy. Each option has tradeoffs: modifying source creates maintenance burden, sidecars add deployment complexity, and log-parsing loses high-cardinality labels like model variant or batch size.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

The single change that mattered most was **adding the `status` label to `inference_requests_total`** — separating successful requests from errors at the Counter level rather than inferring them from HTTP status codes alone. At first, I had a generic `requests_total` counter without a status label, planning to derive errors from the application's HTTP response codes via a separate metric. But during the initial load test, I realized this created a fundamental problem: whenever the app crashed (as it does during `make alert`), HTTP scrape failures meant Prometheus would see *zero data* for the error metric at exactly the moment I needed it most — the hole in data silently suppressed the error rate calculation, making it look like everything was fine when actually the service was down.

By embedding `status="ok"` and `status="error"` directly into the Counter at the application layer (in `main.py`, the `predict()` function explicitly calls `INFERENCE_REQUESTS.labels(model=req.model, status="error").inc()` before raising the HTTPException), the data survives even when the app is partially functional. This connects directly to deck §2's **RED method (Rate, Errors, Duration)** — the insight is that "Errors" should be emitted *before* the error path returns, not after. It also echoes the USE method's principle that every metric should have a clearly defined status dimension so that `rate(errors) / rate(total)` always produces a meaningful ratio, even under degraded conditions. In the SLO burn-rate dashboard, this single design decision makes `inference:fail_ratio:rate5m` a reliable numerator — without it, the multi-window burn-rate alerts (which require both 5m and 1h windows to simultaneously exceed threshold) would have false gaps that break the alerting condition and silently erode our error budget visibility.
