# Investigation playbook

## Query progression

1. Inventory streams by signal type and inspect stream statistics.
2. Read the target schema and full-text search keys.
3. Count or histogram the symptom before pulling individual records.
4. Retrieve a small ordered sample with only useful columns.
5. Pivot on stable identifiers and correlate adjacent signals.

Quote stream and unusual field names with double quotes. Quote string values
with single quotes and escape embedded single quotes by doubling them.

```sql
SELECT COUNT(*) AS errors
FROM "app_logs"
WHERE level = 'error'
```

```sql
SELECT histogram(_timestamp, '5 minute') AS bucket, COUNT(*) AS errors
FROM "app_logs"
WHERE level = 'error'
GROUP BY bucket
ORDER BY bucket
```

```sql
SELECT _timestamp, service, level, message, trace_id
FROM "app_logs"
WHERE trace_id = 'abc123'
ORDER BY _timestamp ASC
```

Use `match_all('term')` only across configured full-text fields. Use
`str_match(field, 'term')` for case-sensitive field matching,
`str_match_ignore_case` for case-insensitive matching, and `re_match` only when
the simpler predicates cannot express the filter.

## Signal correlation

- Logs → traces: pivot on `trace_id`, `span_id`, request ID, service, and time.
- Traces → logs: use the trace start/end plus a small margin and service names.
- Metrics → logs/traces: locate the change point with PromQL, then inspect the
  same interval and dimensions in logs and traces.
- Incidents/alerts → telemetry: extract the rule, stream, labels, and firing
  interval before querying evidence.
- LLM traces: use session/user summary tools, then inspect per-session turns and
  the trace DAG rather than downloading all spans immediately.
- `_timestamp` does not mean the same thing in every signal. A log record is
  usually stamped when it is written (often at completion), while a span is
  stamped at its start, so a slow request can appear in logs well after the
  matching span begins. Pivot on correlation IDs and compare a span's own
  `start_time`/`end_time`; do not read a duration off the log-to-span timestamp
  delta.

Keep all comparisons in UTC and state the exact bounds. A UI display timezone
does not change the API's epoch-microsecond bounds.

## Counting records

One logical operation can emit more than one record. A single MCP request
often produces a protocol-level record and a per-tool record, and a server span
plus its tool span; the duplicates carry different dimensions, so a naive row
count can report roughly double the real number of calls. When you count, filter
on the dimension you mean rather than on rows, and remember a metric family may
expose a bare stream alongside its `_count`/`_sum`/`_bucket` streams.

## Scoping to a shared instance

An organization is not a project. One instance commonly ingests several
projects, folders, or repositories into the same stream — a shared server or
collector tags every call it handles, whoever made it. Before filtering or
aggregating, find the project identity attribute in the schema
(`service_project_path`, `service_project_folder`, or a vendor equivalent),
list its distinct values, and constrain the query to the one you mean:

```sql
SELECT <project_attr>, count(*) AS n FROM "app_logs"
GROUP BY <project_attr> ORDER BY n DESC
```

Without that scope, a count, a percentile, or a top-N silently blends every
project the instance has seen. Prefer a full-path attribute over a
basename-style `*_folder`: the folder is only the directory a process started
in, so two unrelated workloads can share it and one workload can appear under
several folders.

## Cost and precision

- Start narrow; widen only after a no-result check of stream, schema, timezone,
  and units.
- Select columns instead of `SELECT *` after schema discovery.
- Aggregate first when estimating blast radius.
- Prefer JSON for a first pass: rows arrive in `hits`. With
  `agent_options.output_format="csv"` (or `"md_table"`) the server instead puts
  the rows in a single serialized `data` field and leaves `hits` empty, and the
  default `detail="summary"` projection drops `data` — so a CSV query returns
  column names and a correct `total` with no rows. This holds for every query
  shape, not just filtered or ordered ones. Get `data` by asking for
  `detail="full"` (through `tools_call`), or simply keep JSON.
- Report the distribution, not just the mean: p50, p90, p95, and max. When the
  maximum dwarfs p50, name the single outlier and its share of the total, since
  one call can dominate a sum (a 180 s call was 86% of a 208 s tool total). A
  mean alone hides that, and p95 can land below the mean when a single value is
  extreme.
- Use async search jobs only for genuinely long searches; discover their exact
  schemas with `tool_search`.

## Failure handling

- 404: verify `/api/{org}/mcp`, org spelling, base-path proxies, and server
  version.
- 401/403: verify credentials, org membership, and RBAC without printing the
  authorization value. See the credential drift checklist below.
- Missing tool: call `tools/list`, then `tool_search`; do not assume every tool
  is pinned or available in every version.
- Unknown stream/field: rerun `StreamList`/`StreamSchema` and apply any returned
  query suggestions.
- Unknown SQL function: the query engine is DataFusion, not another SQL dialect.
  There is no `datetime()`; render a timestamp with
  `to_timestamp_micros(<micros>)` and bucket with
  `date_trunc('hour', to_timestamp_micros(_timestamp))`. See
  [query-language.md](query-language.md) for the dialect and function catalog.
- Metric streams: each family is its own stream (`_count`, `_sum`, `_bucket`,
  `_min`, `_max`). `StreamList` may also list the bare metric name, but
  `SearchSQL` returns "Search stream not found" for it, so query a family
  stream. Never sum a cumulative `_sum` or `_count` stream, because its rows
  are counters; use PromQL `increase()`/`rate()` for a total, and read outcomes
  from a label filter (for example `outcome = 'error_result'`) rather than from
  a per-row status. A single family can span several services or tools through a
  shared label, so scope it with a label selector before attributing a value to
  one of them, and confirm a `_count` series increases monotonically before
  trusting `rate()`/`histogram_quantile()`, which return `NaN` on a
  non-monotonic or sparse counter.
- Suspected silence: before concluding a source stopped emitting, read the
  stream's own range with `StreamSchema` (`doc_time_min`, `doc_time_max`). A
  narrow query window is indistinguishable from a stalled exporter.
- Empty result: verify signal type, timestamp units, ingest delay, and bounds
  before concluding that no event occurred.
- Empty `hits` with `total > 0`: `total` is the server-side match count; the
  rows matched but were not delivered. The usual cause is a `csv`/`md_table`
  `output_format`, whose rows live in a serialized `data` field that the default
  `detail="summary"` projection omits: retry in JSON (drop `output_format`) or
  request `detail="full"`. Check `from`/`size` pagination before concluding
  anything.
- Zero-looking stream stats: `StreamList`/`StreamSchema` statistics
  (`doc_num`, `storage_size`) can report 0 on freshly created or stats-lagged
  streams that actually hold data. Confirm emptiness with a small bounded
  count query, never with the stats block.
- Timeout/large scan: narrow bounds or predicates, select fewer fields, then use
  server-side partition mode.

## Pipeline-derived fields

`severity`, `level`, and similar classification fields are typically not
emitted by the application — they are assigned downstream by the shipping
pipeline (an OpenTelemetry Collector transform, Fluent Bit parser, or
OpenObserve ingestion rule), often by keyword-matching the message text.
Keyword matching false-positives in both directions: an INFO line quoting
`span_status = 'ERROR'` or fetching `useErrorTracking.js` gets tagged ERROR,
while a genuinely severe message without the keyword stays "unspecified".

Before filtering or counting on such a field, profile it and spot-check:

```sql
SELECT severity, COUNT(*) AS n FROM "app_logs" GROUP BY severity ORDER BY n DESC
```

```sql
SELECT _timestamp, severity, body FROM "app_logs"
WHERE severity = 'ERROR'
ORDER BY _timestamp DESC
```

If a sample of flagged rows does not actually carry that level in `body`, the
field is keyword-derived: report the misclassification, and build the
investigation on anchored body patterns instead — match the level token where
the known log format places it (for example `... UTC [151] ERROR:` or
`...T15:19:15Z\tERROR\t`), not a bare substring anywhere in the message.
A uniform `severity = 0`/`unspecified` distribution is the same finding with
the opposite sign: no classifier is running at all.

## Credential drift

A credential that worked earlier in the session and now returns 401 is
usually drift between the places the secret is stored, not a server fault.
For compose-based local instances check, in order:

1. File consistency: the ready-made header matches the derived one —
   `OPENOBSERVE_AUTH` equals `base64(ROOT_USER_EMAIL:ROOT_USER_PASSWORD)`
   from the same file. OpenObserve applies `ZO_ROOT_USER_*` at container
   startup, so the derived pair is authoritative.
2. Resolved configuration: `docker compose config` shows what containers
   actually receive; compose prefers exported process environment over
   `.env`, so a stale shell export shadows file edits.
3. Running containers: they snapshot environment at create time. Compare
   `docker inspect <container> --format '{{.Config.Env}}'` and recreate
   (`docker compose up -d --force-recreate <service>`) after credential
   changes. Prefer `docker inspect` over `docker exec ... printenv`; exec
   output can carry noise that mimics a wrong value.

Verify secrets without exposing them — printing, decoding, or diffing a live
credential may be blocked by host permission classifiers and must never
appear in a report. Compare equality instead:

```bash
EXPECTED=$(grep '^OPENOBSERVE_AUTH=' .env | cut -d= -f2- | tr -d '"')
DERIVED="Basic $(printf '%s' "$EMAIL:$PASSWORD" | base64)"
[ "$EXPECTED" = "$DERIVED" ] && echo MATCH || echo MISMATCH
```

SHA-256 fingerprint comparison (`shasum -a 256`) works the same way when
the two values live in different shells or machines; value-length checks
catch truncation. Report only MATCH/MISMATCH, never the compared material.

## Evidence report

Record the incident question, UTC time range, organization, signal type,
streams, exact query or tool call, relevant result counts/samples, and any
blind spots. Separate observations, inferences, and recommended next checks.
