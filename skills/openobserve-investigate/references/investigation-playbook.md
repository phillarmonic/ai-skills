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

Keep all comparisons in UTC and state the exact bounds. A UI display timezone
does not change the API's epoch-microsecond bounds.

## Cost and precision

- Start narrow; widen only after a no-result check of stream, schema, timezone,
  and units.
- Select columns instead of `SELECT *` after schema discovery.
- Aggregate first when estimating blast radius.
- Use CSV for tabular hits and summary detail for discovery.
- Use async search jobs only for genuinely long searches; discover their exact
  schemas with `tool_search`.

## Failure handling

- 404: verify `/api/{org}/mcp`, org spelling, base-path proxies, and server
  version.
- 401/403: verify credentials, org membership, and RBAC without printing the
  authorization value.
- Missing tool: call `tools/list`, then `tool_search`; do not assume every tool
  is pinned or available in every version.
- Unknown stream/field: rerun `StreamList`/`StreamSchema` and apply any returned
  query suggestions.
- Empty result: verify signal type, timestamp units, ingest delay, and bounds
  before concluding that no event occurred.
- Timeout/large scan: narrow bounds or predicates, select fewer fields, then use
  server-side partition mode.

## Evidence report

Record the incident question, UTC time range, organization, signal type,
streams, exact query or tool call, relevant result counts/samples, and any
blind spots. Separate observations, inferences, and recommended next checks.
