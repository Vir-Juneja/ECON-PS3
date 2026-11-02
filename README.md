# ECON-PS3

I've created `review_query(con, sql)` as a reusable SQL quality check for DuckDB. It returns structured warning dictionaries, not printed messages, and uses only the Python standard library and DuckDB.

Download review_query.py

### What it detects

| Warning code                 | Detection                                                         |
| ---------------------------- | ----------------------------------------------------------------- |
| `JOIN_CARDINALITY_CHANGE`    | Join duplicates or loses rows relative to the starting table      |
| `NULL_FILTER_DROPS_UNKNOWNS` | `WHERE` evaluates to `UNKNOWN`, silently excluding rows           |
| `NULLABLE_FILTER_RISK`       | A nullable comparison could exclude future null values            |
| `AGGREGATE_ON_FANOUT`        | `SUM`, `COUNT`, `AVG`, etc. operate on duplicated base-table rows |
| `REVIEW_INCOMPLETE`          | Query structure exceeds the reviewer's supported scope            |

### How to use it

```
from review_query import review_querysql = """SELECT SUM(c.budget) AS total_budgetFROM customers cLEFT JOIN orders o    ON c.customer_id = o.customer_idWHERE o.status <> 'cancelled'"""warnings = review_query(con, sql)codes = {w["code"] for w in warnings}assert "JOIN_CARDINALITY_CHANGE" in codesassert "NULL_FILTER_DROPS_UNKNOWNS" in codesassert "AGGREGATE_ON_FANOUT" in codes
```

Every warning contains `code`, `severity`, `message`, and a `details` dictionary with measurements or predicates suitable for assertions.

### Mechanism checks

Fan-out: The function counts joined rows and distinct starting-table identities in the same aggregate probe. It also counts the starting table separately. This reveals duplicated and missing rows without executing the complete join twice. DuckDB supports the conditional aggregate `FILTER` clauses used in the probe.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://duckdb.org\&sz=32)

DuckDB

+1



Three-valued logic: SQL `WHERE` retains only `TRUE`. A comparison such as `status <> 'cancelled'` evaluates to `UNKNOWN` for null statuses, which means these rows are discarded even though they are not known to be cancelled. The reviewer measures how many rows are excluded this way.

### Evaluation of the $1,240

The file includes two illustrative queries using `customers` and `orders`, with executable assertions.

Illustrative joined SUM

# $1,240

One customer amount counted twice

Correct customer-grain SUM

# $1,140

Each customer's amount counted once

The first illustrative query is designed to trigger `JOIN_CARDINALITY_CHANGE` and `AGGREGATE_ON_FANOUT`. The second additionally triggers `NULL_FILTER_DROPS_UNKNOWNS`.

Would it have caught your actual $1,240? If that figure arose from summing customer-level amounts after a one-to-many orders join, the reviewer is designed to flag the problem before dashboard release.

I cannot verify your two actual Phase 1 queries because neither their SQL nor their database is available here. DuckDB also wasn't installed in this execution environment, so I checked the probe logic with a compatible SQL engine rather than claiming a DuckDB execution. The downloadable file includes DuckDB assertions you can run directly with `python review_query.py`.

The reviewer is deliberately scoped to straightforward SELECT/JOIN statements. For unsupported constructions such as CTEs and derived-table joins, it returns `REVIEW_INCOMPLETE` instead of silently certifying a query as safe.
