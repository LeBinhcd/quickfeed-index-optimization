# QuickFeed index optimization report

## Diagnosis

The legacy `Posts` table had five secondary indexes in addition to its primary key. An `INSERT` must write the row and update every affected secondary B-Tree, including page navigation, possible page splits, and durable redo-log work. Therefore, unnecessary indexes increase write latency, RAM pressure, and disk usage.

`idx_content` is a prefix B-Tree over a long `TEXT` value. It consumes storage and is not a general full-text search solution. `idx_post_type` has only three likely values, and `idx_is_visible` has only two. Their low cardinality often makes a table scan cheaper than repeated index lookups, especially when most rows are visible. The optimizer may therefore ignore them while writes still pay their maintenance cost.

The optimization keeps `idx_user_id`, which supports retrieving a user's posts, and `idx_created_at`, which supports time-based newsfeed access. It removes exactly `idx_content`, `idx_post_type`, and `idx_is_visible`. This favors write performance and reduces index maintenance while preserving the most useful access paths. If keyword search in `content` is required, a deliberate `FULLTEXT` design should be evaluated instead of an indiscriminate prefix index.

## Measurement log

The script was executed locally against MySQL 8.0.46 with approximately 10,000 sample rows.

| Measurement | Data (MB) | Index (MB) | Total (MB) |
|---|---:|---:|---:|
| Before optimization | 0.02 | 0.08 | 0.09 |
| After optimization | 0.02 | 0.08 | 0.09 |

The `SHOW INDEX FROM Posts` result after optimization confirmed that only these indexes remained:

```text
PRIMARY
idx_user_id
idx_created_at
```

The displayed `INDEX_LENGTH` did not decrease for this small test table because InnoDB reports allocated index pages and the values are rounded to two decimal megabytes. This does not mean the three indexes remained: their definitions were removed successfully. On a larger populated table, the storage difference is more likely to be visible. In production, also measure filesystem usage and consider table rebuild/reorganization policies when physical space reclamation is required.

## Trade-off and limitation

Removing indexes can make some selective reads slower. The correct decision depends on query workload, cardinality, selectivity, and measured plans. For an append-only archive, extra indexes may be acceptable because write maintenance is rare, but they still consume disk and affect backup and cache requirements. `INDEX_LENGTH` is an allocated storage statistic and may not immediately shrink at the filesystem level after dropping an index; verify database-reported value and operational disk usage separately.
