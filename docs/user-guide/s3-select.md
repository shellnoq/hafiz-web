# S3 Select — SQL on objects (with types that actually work)

`SelectObjectContent` runs a SQL `SELECT` directly against the bytes of a CSV or JSON object, so the client never has to download the whole file just to find a few rows. Hafiz ships a subset compatible with the AWS API plus one improvement that no other S3 implementation has: **schema-aware type coercion**.

## Basic query

```bash
aws --endpoint-url http://hafiz:9000 s3api select-object-content \
  --bucket datalake \
  --key 2026/events.csv \
  --expression-type SQL \
  --expression "SELECT s.user_id, s.event FROM S3Object s WHERE s.event = 'purchase' LIMIT 100" \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE"}}' \
  --output-serialization '{"CSV":{}}' \
  /dev/stdout
```

## Why schema-aware matters

A classic bug in every S3 implementation that ignores column types:

```sql
SELECT name FROM S3Object WHERE age > '20'
```

When the query engine compares everything as strings, `"100" > "20"` is **false** because `'1'` sorts before `'2'`. Legitimate rows silently disappear from the result set.

Hafiz runs `hafiz_catalog::infer()` on the first few MB of the object before query execution. If the `age` column is inferred as `Int` or `Float`, the string literal `'20'` is promoted to the number `20` and the comparison uses numeric ordering. The row with `age=100` matches; the row with `age=7` doesn't.

| Column type | Literal | Behavior |
|---|---|---|
| `Int` / `Float` | `'20'` or `20` | both treated as numbers |
| `String` | `'Alice'` or `Alice` | both treated as strings |
| `Bool` | `true`, `false`, `1`, `0` | parsed as boolean |
| `Date` | RFC 3339 / `YYYY-MM-DD` | string compare (numeric is on roadmap) |

## Supported SQL subset

- `SELECT col_list FROM S3Object [alias] [WHERE …] [LIMIT n]`
- Operators: `=`, `!=`, `<>`, `<`, `>`, `<=`, `>=`, `LIKE`, `IS NULL`, `IS NOT NULL`
- Logical: `AND`, `OR`
- `LIKE` patterns: `%` for any, `_` for one character
- Column references: `col`, `s.col`, `_1`, `_2` (positional for headerless CSV)

## Supported input formats

| Format | Example serialization |
|---|---|
| CSV with header | `{"CSV":{"FileHeaderInfo":"USE"}}` |
| Headerless CSV | `{"CSV":{"FileHeaderInfo":"NONE"}}` (refer to columns as `_1`, `_2`, …) |
| JSON Document | `{"JSON":{"Type":"DOCUMENT"}}` — single object or top-level array |
| JSON Lines | `{"JSON":{"Type":"LINES"}}` — one object per line |

Output is CSV (default) or JSON array.

## Compressed input

Hafiz transparently decompresses `x-hafiz-compression-algorithm: zstd` objects before running the query, so you can store datasets compressed on disk without teaching the SQL engine about the format.

## Related

- [Data Catalog](../architecture/components.md#data-catalog) — the schema inference layer Select uses; exposed via an admin endpoint for discovery on top of ad-hoc bucket contents.
- [Change Stream / EventBridge](event-bridge.md) — stream notifications when query-relevant objects land.
