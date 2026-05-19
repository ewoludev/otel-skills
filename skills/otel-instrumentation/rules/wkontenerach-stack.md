---
title: 'Ewolucja Developera local stack: Prometheus, Tempo, Loki, Grafana'
impact: HIGH
tags:
  - wkontenerach
  - ewolucja-developera
  - local-dev
  - prometheus
  - tempo
  - loki
  - no-collector
---

# wKontenerach local stack

This rule overrides the default OTLP exporter configuration in [resolve-values](./resolve-values.md) and the SaaS-oriented examples in the language SDK rules (see [./sdks/](./sdks/)) when the target deployment is the local stack from module 5 of the "Ewolucja Developera" course.

When this rule applies, the application sends telemetry **directly** to a local stack of Prometheus, Tempo, Loki, and Grafana — no OpenTelemetry Collector is involved.

This rule defines the **transport contract** between the application and the stack (endpoints, protocols, formats).
The language-specific SDK setup lives in the rule for the target SDK under [./sdks/](./sdks/) (e.g., [nodejs](./sdks/nodejs.md), [python](./sdks/python.md), [java](./sdks/java.md), [go](./sdks/go.md), [dotnet](./sdks/dotnet.md)).
Code snippets in this file use Node.js as the illustrative example because the course demo application is Node; the equivalent setup for any other SDK must follow the matching `./sdks/<language>.md` rule.

## When to use this rule

Apply this rule when **all** of the following hold:

1. The project contains a `docker-compose.yml` (or equivalent) — at the repository root or under `infra/`, `stack/`, or `observability/` — that declares services named `tempo`, `loki`, and `prometheus`.
2. There is **no** `otel-collector` or `otelcol` service in the Compose file.
3. The default exporter values resolved per [resolve-values](./resolve-values.md) point at a SaaS endpoint (e.g., `https://*.dash0.com`), or no exporter values exist yet.

If even one condition is false, follow [resolve-values](./resolve-values.md) and the relevant SDK rule instead.

## Stack overview

| Signal | Source | Transport | Destination |
| --- | --- | --- | --- |
| Traces | Application SDK | OTLP HTTP (port 4318) | `tempo` container |
| Metrics | Application `/metrics` endpoint | Pull (scrape) | `prometheus` container |
| Logs | Application stdout (JSON) | Docker JSON log files (file scrape) | `alloy` → `loki` container |

No authentication is required.
All endpoints are unauthenticated and only reachable on the local Docker network or `localhost`.

## Traces: OTLP HTTP directly to Tempo

Set the following environment variables instead of the values resolved per [resolve-values § OTLP endpoint](./resolve-values.md#otlp-endpoint).
These variables are SDK-agnostic and apply identically to every OpenTelemetry language SDK:

```bash
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4318/v1/traces
```

When the application container shares the Docker network with the stack, replace `localhost` with the Compose service alias `tempo`:

```bash
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://tempo:4318/v1/traces
```

Do not set `OTEL_EXPORTER_OTLP_HEADERS`.
Tempo on this stack does not require authentication, and an unexpected `Authorization` header causes Tempo to return 401 and drop the spans silently from the SDK's perspective.

Do not set the unqualified base variable `OTEL_EXPORTER_OTLP_ENDPOINT`.
This stack uses signal-specific endpoints so that the traces pipeline (OTLP) and the metrics pipeline (Prometheus pull on `/metrics`) stay separately routable — see the next section.

## Metrics: Prometheus pull endpoint

The course stack scrapes metrics — it does **not** receive OTLP metric pushes.
Disable the OTLP metrics exporter and add the Prometheus pull exporter to the SDK.

### Step 1 — disable OTLP metrics

This environment variable is SDK-agnostic:

```bash
OTEL_METRICS_EXPORTER=none
```

Without this, the SDK attempts to push metrics to the OTLP endpoint configured for traces.
Tempo accepts the connection but silently drops metrics — the dashboard stays empty and no error surfaces.

### Step 2 — add the Prometheus exporter

Install and activate the Prometheus pull exporter that ships with the target SDK.
The exporter must:

- Start an HTTP server bound to all interfaces on **port 9464**.
- Serve metrics in Prometheus text format at the path **`/metrics`**.
- Be registered with the `MeterProvider` *before* any code path records metrics.

The package name, registration API, and activation entrypoint differ per SDK — follow the matching rule under [./sdks/](./sdks/).

**Example for Node.js**:

Install the package:

```bash
npm install @opentelemetry/exporter-prometheus
```

Activate the exporter at the application entrypoint, before any code path that records metrics:

```js
const { MeterProvider } = require('@opentelemetry/sdk-metrics');
const { PrometheusExporter } = require('@opentelemetry/exporter-prometheus');

const prometheusExporter = new PrometheusExporter({
  port: 9464,
  endpoint: '/metrics',
});

const meterProvider = new MeterProvider({
  readers: [prometheusExporter],
});
```

Pair this with the auto-instrumentation activation (`--require @opentelemetry/auto-instrumentations-node/register` for CommonJS or `--import` for ESM) per [nodejs § activate the SDK](./sdks/nodejs.md#1-activate-the-sdk).

For other SDKs (Python, Java, Go, .NET, and so on), use the equivalent Prometheus exporter package and registration API documented in the corresponding `./sdks/<language>.md` rule.

### Step 3 — configure the scrape target

The course stack's `prometheus.yml` scrape config must target the application:

- When the application runs on the host: `host.docker.internal:9464`.
- When the application runs inside the Compose network: `<app-service-name>:9464` where `<app-service-name>` is the Compose service name of the application.

### Step 4 — keep metric cardinality bounded

Pull scrapes load every active series into Prometheus memory on each tick.
Follow [metrics § cardinality management](./metrics.md#cardinality-management) strictly — high-cardinality attributes (`user.id`, `url.full`) crash the local stack within minutes.

## Logs: stdout JSON for Alloy

The course stack uses Grafana Alloy to read application logs from the Docker JSON log files on disk (`/var/lib/docker/containers/*/*-json.log`).
Application code emits **structured JSON to stdout** — it does **not** export logs via OTLP.

### Step 1 — disable the OTLP logs exporter

This environment variable is SDK-agnostic:

```bash
OTEL_LOGS_EXPORTER=none
```

### Step 2 — emit structured JSON on stdout

Configure the application's logging library to write one JSON record per line to stdout.
Alloy parses each line through a `loki.process` component with a `stage.json` block.
Because Docker wraps every stdout line in its own JSON envelope (`{"log":"...","stream":"stdout","time":"..."}`), Alloy must apply the `stage.json` block twice: once to extract the inner `log` field from the Docker envelope, then again on that field to parse the application's structured record.
The exact library and serializer setup is SDK-specific — see the `structured logging` section in the matching rule under [./sdks/](./sdks/).

**Example for Node.js**: use pino with its default JSON output, which already emits one structured JSON record per line on stdout.
Follow [nodejs § structured logging](./sdks/nodejs.md#structured-logging) for the serializer setup, but skip the OTLP logs bridge.

For other SDKs, use the canonical structured logger for the language (e.g., `structlog` or the standard `logging` module with a JSON formatter for Python, `Logback` with `logstash-logback-encoder` for Java, `slog` with a JSON handler for Go, `Serilog` with `Serilog.Formatting.Compact` for .NET).

### Step 3 — inject trace correlation manually

Auto-instrumentation does not propagate `trace_id` and `span_id` into log records when the OTLP logs exporter is disabled.
Add a hook to the logger that, on every record, reads the active span from the OpenTelemetry API and writes `trace_id` and `span_id` as top-level JSON fields.
The hook mechanism differs per SDK — see the matching rule under [./sdks/](./sdks/).

**Example for Node.js** using pino's `mixin` callback:

```js
const pino = require('pino');
const { trace } = require('@opentelemetry/api');

const logger = pino({
  mixin() {
    const span = trace.getActiveSpan();
    if (!span) return {};
    const ctx = span.spanContext();
    return { trace_id: ctx.traceId, span_id: ctx.spanId };
  },
});
```

For other SDKs, use the equivalent extension point: a `structlog` processor for Python, a `MDC`-backed JSON encoder for Java/Logback, a custom `slog.Handler` for Go, or a `Serilog` enricher for .NET.

Alloy's `loki.process` component in the course stack must include a `stage.json` block that parses the record and a `stage.labels` block that promotes `trace_id` and `span_id` to Loki labels.
The full Alloy pipeline is: `loki.source.file` (reads `/var/lib/docker/containers/*/*-json.log`) → `loki.process` (Docker envelope + application JSON + labels) → `loki.write` (`http://loki:3100/loki/api/v1/push`).
This enables click-through from a Tempo span to the matching Loki log line in Grafana.

> [!CAUTION]
> Keep `trace_id` and `span_id` out of Loki labels in any deployment with non-trivial traffic — every unique value creates a new log stream and explodes Loki's index.
> The course stack accepts the cardinality cost in exchange for one-click trace↔log correlation.
> For production, drop these as labels and use `loki.process`'s `stage.structured_metadata` block instead, then query with `| trace_id="..."` (a non-indexed filter).

## Complete `.env.otel` example

The variables below are SDK-agnostic and apply to every language SDK:

```bash
# Service identity — resolve per resolve-values rule
OTEL_SERVICE_NAME=my-app
OTEL_RESOURCE_ATTRIBUTES=deployment.environment.name=local

# Traces → Tempo (direct OTLP HTTP, no Collector)
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4318/v1/traces

# Metrics → Prometheus pull (programmatic SDK setup, disable OTLP push)
OTEL_METRICS_EXPORTER=none

# Logs → stdout JSON for structured logger (no OTLP)
OTEL_LOGS_EXPORTER=none
```

Activation of auto-instrumentation is SDK-specific.
**Example for Node.js** — append the activation flag and run with `--env-file`:

```bash
# Activate auto-instrumentation (Node.js)
NODE_OPTIONS=--require @opentelemetry/auto-instrumentations-node/register
```

```bash
node --env-file=.env.otel app.js
```

The `--env-file` flag requires Node.js 20.6 or later.

For other SDKs, replace the activation line with the equivalent mechanism from the matching `./sdks/<language>.md` rule (e.g., `opentelemetry-instrument` for Python, the `-javaagent` flag for Java, `dotnet-monitor` or the autoinstrumentation NuGet package for .NET).

## Validation

After applying this rule, verify each signal independently in Grafana:

1. **Traces** — open Grafana → Explore → Tempo and run `{ service.name = "<your service>" }`.
   A request handled by the application in the last 5 minutes must appear as a trace.
2. **Metrics** — open Grafana → Explore → Prometheus and run `up{job="<your job>"}`.
   The value must be `1`.
   Then run the auto-instrumented HTTP counter for your stack (e.g., `http_server_request_duration_seconds_count`).
   A non-zero value confirms the scrape works.
3. **Logs** — open Grafana → Explore → Loki and run `{service_name="<your service>"} | json`.
   The most recent application log must be parsed as JSON and surface `trace_id` and `span_id` as labels.

If any signal is missing, run the [validation](./validation.md) checklist for that signal type to triage.

## What this rule does not cover

- **Production deployments.**
  This rule targets local development on a single machine.
  For production observability, deploy an OpenTelemetry Collector and follow [otel-collector](../../otel-collector/SKILL.md) instead of the direct SDK-to-backend pattern documented here.
- **Tail sampling, span-derived RED metrics, and attribute enrichment.**
  These require a Collector — see [sampling](../../otel-collector/rules/sampling.md) and [red-metrics](../../otel-collector/rules/red-metrics.md).
- **Kubernetes deployment.**
  For cluster-based stacks, follow [k8s](./platforms/k8s.md).
- **Per-SDK implementation details.**
  This rule defines only the transport contract (env vars, scrape target, JSON log format, trace correlation requirement).
  The Prometheus exporter activation, structured logger setup, and trace-correlation hook for each language SDK live in the matching rule under [./sdks/](./sdks/).
