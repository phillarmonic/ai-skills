# OpenObserve query language

OpenObserve's query engine is Apache DataFusion, and its documentation states
that all supported SQL syntax and functions are available through DataFusion.
Treat the DataFusion SQL reference as the baseline, with a layer of
OpenObserve-specific functions on top.

- Engine: [Apache DataFusion](https://datafusion.apache.org/user-guide/sql/index.html)
- [OpenObserve SQL reference](https://openobserve.ai/docs/reference/sql-reference)
- [Function categories](https://openobserve.ai/docs/reference/sql-functions)

Searches are read-only. `SELECT` is the only statement shape the search API
accepts; there is no DDL or DML against telemetry.

## Statement shape

```sql
[WITH cte [, ...]]
SELECT [DISTINCT | DISTINCT ON (expr)] select_list
FROM "stream" | (subquery)
[JOIN ...]
[WHERE predicate]
[GROUP BY ...] [HAVING ...]
[ORDER BY ... [NULLS FIRST | NULLS LAST]]
[LIMIT n] [OFFSET n]
```

Also available: subqueries, CTEs including `WITH RECURSIVE`,
`UNION`/`INTERSECT`/`EXCEPT`, `INNER`/`LEFT`/`RIGHT`/`FULL`/`CROSS` joins,
window functions, and the `QUALIFY` clause.

```sql
WITH slow AS (
  SELECT operation_name, duration
  FROM "app_logs"
  WHERE duration > 1000000
)
SELECT operation_name, count(*) AS n, round(avg(duration) / 1000.0, 1) AS avg_ms
FROM slow
GROUP BY operation_name
HAVING count(*) > 1
ORDER BY avg_ms DESC
LIMIT 10
```

## Identifiers

- The parser lowercases unquoted identifiers, but inferred schema field names
  are not changed. Any field with a capital letter or an unusual character must
  be double-quoted.
- Quote stream names: `FROM "app_logs"`.
- String literals use single quotes; escape an embedded quote by doubling it.

## Time

- `_timestamp` is event time in epoch microseconds; `_o2_ingest_ts` is ingestion
  time in the same unit.
- Search `start_time` and `end_time` are epoch microseconds and cannot be zero.
- Bucket with `histogram` (OpenObserve) or `date_bin` (DataFusion):

```sql
SELECT histogram(_timestamp, '5 minute') AS bucket, count(*) AS n
FROM "app_logs" GROUP BY bucket ORDER BY bucket
```

```sql
SELECT date_bin(interval '5 minutes', to_timestamp_micros(_timestamp)) AS bucket,
       count(*) AS n
FROM "app_logs" GROUP BY bucket LIMIT 3
```

- `histogram` needs an explicit duration (`'30 seconds'`, `'1 minute'`,
  `'15 minutes'`, `'1 hour'`). Without one the bucket size follows an internal
  default.
- Render or floor a timestamp with `to_timestamp_micros(<epoch micros>)` and
  `date_trunc('hour', to_timestamp_micros(_timestamp))`.
- There is no `datetime()` function and no `DATETIME` type. DataFusion lists
  `DATETIME` among its unsupported SQL types, and a call returns
  `unknown function 'datetime'`.

## Full-text search

| Function | Behaviour |
| --- | --- |
| `match_all('term')` | Keyword across fields configured for full-text search. Case-insensitive. Supports `ab*`, `*ab`, `*ab*`, and `key1 key2*`. |
| `match_all_raw('term')` | Same fields with a raw scan. Case-sensitive; `match_all_raw_ignore_case` for the other case. |
| `str_match(field, 'value')` | Substring in one field. Case-sensitive. Alias `match_field`. |
| `str_match_ignore_case(field, 'value')` | Substring in one field, case-insensitive. Alias `match_field_ignore_case`. |
| `re_match(field, 'pattern')` | Rust `regex` syntax on one field. Case-sensitive unless the pattern opts in with `(?i)`. |
| `re_not_match(field, 'pattern')` | Inverse of `re_match`. |

- Default full-text fields are `log`, `message`, `msg`, `content`, `data`,
  `body`, `json`, `error`, `llm_input`, and `llm_output`. More can be enabled in
  stream settings.
- Inverted-index behaviour requires `ZO_ENABLE_INVERTED_INDEX=true` plus
  per-field configuration.
- `str_match` on a single field is more efficient than `match_all`.
- `match_all` requires the stream to have full-text search fields. A stream
  without them is rejected outright rather than returning no rows:

```json
{"code":20001,"message":"Search SQL not valid",
 "error_detail":"match_all() should only apply to the stream that have full text search fields"}
```

Trace streams in particular often have no full-text fields, so prefer
`str_match` or `re_match` on an attribute when the stream is traces.

## Null and numeric filtering

- Use `field is null` and `field is not null`. `field = ''` and `field != ''` do
  not behave correctly for numeric fields.
- Use SQL operators, so `>=` rather than `=>`.
- Compare boolean fields unquoted, for example `success = false`. Quoting the
  literal makes the planner attempt a coercion and fail with
  `Cannot infer common argument type for comparison operation Boolean = Utf8`.

## Aggregates

`COUNT`, `SUM`, `AVG`, `MIN`, and `MAX` follow PostgreSQL semantics, and
`count(case when <pred> then 1 end)` is the idiomatic conditional count.

```sql
SELECT count(case when status_code <> 0 then 1 end) AS errors, count(*) AS total
FROM "app_logs"
```

Percentiles and top-K are approximate:

| Function | Purpose |
| --- | --- |
| `approx_percentile_cont(field, q)` | Continual percentile, for example `approx_percentile_cont(duration, 0.95)`. |
| `approx_topk(field, k)` | Top K most frequent values. Space-Saving, so a globally frequent value can be missed. |
| `approx_topk_distinct(field1, field2, k)` | Top K of `field1` by distinct count of `field2`. HyperLogLog. |

`approx_topk` and `approx_topk_distinct` return an object of rows and are
normally unwrapped with `unnest`:

```sql
SELECT item.clientip AS clientip, item.request_count AS request_count
FROM (SELECT unnest(approx_topk(clientip, 10)) AS item FROM "default")
ORDER BY request_count DESC
```

No filter clause applies to `approx_topk_distinct`.

## Arrays and JSON

Array fields are usually stored as stringified JSON (`["a","b"]`). OpenObserve
provides functions for that form: `arrcount`, `arrsort`, `arr_descending`,
`arrjoin`, `arrindex`, `arrzip`, and `spath(field, 'a.b')` for a nested JSON
value.

Native DataFusion array functions (`unnest`, `array_union`, `array_concat`,
`array_pop_back`) do not accept the stringified form. Convert first with
`cast_to_arr(field)`, and convert back with `to_array_string(...)` to reuse the
OpenObserve array functions.

## Metrics

Metric families are queried with PromQL, which OpenObserve describes as about
97% compliant, not with SQL. Histogram and counter families are also exposed as
their own streams (`_count`, `_sum`, `_bucket`, `_min`, `_max`) which SQL can
query, subject to the cumulative-counter caveat in the investigation playbook.

A family is shared across everything that emits it, so scope it with a label
selector before reading one service's or tool's values. A `_count`/`_sum`
series is cumulative: confirm it increases monotonically before relying on
`rate()` or `histogram_quantile()`, which return `NaN` on a non-monotonic or
sparse counter.

## Verified behaviours

Confirmed against a live instance:

- `histogram(_timestamp, '5 minute')`, `date_bin(interval '5 minutes', ...)`,
  and `to_timestamp_micros(...)` all work.
- `count(case when ... then 1 end)` works.
- A quoted boolean (`success = 'false'`) fails type coercion; the unquoted form
  (`success = false`) works.
- `str_match(field, 'value')` works against a trace attribute.
- `match_all(...)` against a trace stream fails with the full-text-field error
  shown above.
- `datetime(...)` fails with `unknown function 'datetime'`.
