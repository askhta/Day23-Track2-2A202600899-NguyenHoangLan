# Day 23 Lab Reflection

**Student:** Nguyen Hoang Lan  
**Submission date:** 2026-06-29  
**Lab repo URL:** local submission, update with public GitHub URL before final upload

---

## 1. Hardware + setup output

Output of `python 00-setup/verify-docker.py` in this workspace:

```text
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 7.63 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: D:\VinAi\Phase8\Day23-Track2-2A202600899-NguyenHoangLan\00-setup\setup-report.json
```

The setup report is committed at `00-setup/setup-report.json`. Ports were reported as bound because the observability stack was already running when the report was refreshed.

---

## 2. Track 02 - Dashboards & Alerts

### 6 essential panels

Expected evidence file: `submission/screenshots/dashboard-overview.png` after `make load`.

### Burn-rate panel

Expected evidence file: `submission/screenshots/slo-burn-rate.png` after enough traffic has accumulated for the burn-rate windows.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | Prometheus target went down |
| T0+120s | `ServiceDown` fired | Alertmanager showed 1 active alert |
| T1 | started `day23-app` | app health returned |
| T1+60s | alert resolved | Prometheus rule returned to inactive |

Slack delivery requires replacing the placeholder webhook with a real Slack incoming webhook before capturing Slack screenshots.

### One thing surprised me about Prometheus / Grafana

The most important detail was label discipline. The app exposes model and status labels for request counters, but avoids high-cardinality labels such as prompt text or request ID. That keeps Prometheus useful for SLO math instead of turning it into an expensive event store.

---

## 3. Track 03 - Tracing & Logs

### One trace screenshot from Jaeger

Jaeger retained trace `8318bf4dcf723aa786733ae3ade3b118`, showing the `POST /predict` route and the manual child spans `predict`, `embed-text`, `vector-search`, and `generate-tokens`. The request used `slow: true` so the tail-sampling policy kept it through the `keep-slow` branch.

### Log line correlated to trace

Example structured log shape emitted by the app:

```json
{"model":"llama3-mock","input_tokens":4,"output_tokens":54,"quality":0.848,"duration_seconds":2.2849,"trace_id":"8318bf4dcf723aa786733ae3ade3b118","event":"prediction served","level":"info","timestamp":"2026-06-29T05:37:37.232477Z"}
```

The `trace_id` field is deliberately included in the JSON log so Grafana Loki can use the configured derived field to link from a log line to the same trace in Jaeger.

### Tail-sampling math

The collector policy keeps 100% of traces with `ERROR` status, 100% of traces slower than 2 seconds, and 1% of the remaining healthy traces. If the service produces 100 traces/sec with 2 error traces/sec, 1 slow healthy trace/sec, and 97 normal healthy traces/sec, the retained rate is `2 + 1 + 0.01 * 97 = 3.97 traces/sec`, or about 3.97% overall. This gives complete evidence for incidents while reducing storage for normal traffic.

---

## 4. Track 04 - Drift Detection

### PSI scores

Output of `04-drift-detection/reports/drift-summary.json`:

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

The HTML report is available at `04-drift-detection/reports/drift-report.html`. In this environment Evidently was unavailable, so the script generated a fallback HTML report from the same PSI/KL/KS metrics.

### Which test fits which feature?

For `prompt_length`, I would use PSI as the primary production signal because it is easy to threshold and explain for a shifted continuous operational feature. For `embedding_norm`, I would use KS for a one-dimensional continuous distribution and add MMD when comparing full embedding vectors instead of only their norm. For `response_length`, PSI works well for monitoring bucketed serving behavior, while KS is useful during investigation because it tests the full distribution. For `response_quality`, I would monitor PSI for dashboarding and KL divergence for sensitivity to a large shape change in the score distribution.

---

## 5. Track 05 - Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day metric is the Day 20 model-serving metric from llama.cpp because it depends on how the serving process exposes Prometheus metrics and whether it is reachable from inside Docker via `host.docker.internal`. Qdrant-style metrics are usually already Prometheus-friendly, but model-serving endpoints often need command-line flags, port mapping, and naming cleanup before they are useful in a shared dashboard.

---

## 6. The single change that mattered most

The single change that mattered most was keeping RED metrics, AI-specific metrics, and trace correlation separate but linkable. Request count, latency histogram, and error status are enough for SLO burn-rate alerts; token counters and quality gauge explain AI cost and output behavior; trace IDs in logs connect a user-visible failure to a concrete execution path. That separation turns the stack from a collection of charts into an operations tool.

This matches the deck's RED/USE plus fourth-pillar framing: generic service health tells us whether the API is alive, while token cost and quality tell us whether the AI service is still useful. The useful design choice was not adding more metrics; it was choosing metrics that answer different operational questions without creating high-cardinality noise.
