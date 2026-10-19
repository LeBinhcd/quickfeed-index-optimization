# QuickFeed index optimization report

## Diagnosis

The legacy `Posts` table had five secondary indexes in addition to its primary key. An `INSERT` must write the row and update every affected secondary B-Tree, including page navigation, possible page splits, and durable redo-log work. Therefore, unnecessary indexes increase write latency, RAM pressure, and disk usage.

`idx_content` is a prefix B-Tree over a long `TEXT` value. It consumes storage and is not a general full-text search solution. `idx_post_type` has only three likely values, and `idx_is_visible` has only two. Their low cardinality often makes a table scan cheaper than repeated index lookups, especially when most rows are visible. The optimizer may therefore ignore them while writes still pay their maintenance cost.

The optimization keeps `idx_user_id`, which supports retrieving a user's posts, and `idx_created_at`, which supports time-based newsfeed access. It removes exactly `idx_content`, `idx_post_type`, and `idx_is_visible`. This favors write performance and reduces index storage while preserving the most useful access paths. If keyword search in `content` is required, a deliberate `FULLTEXT` design should be evaluated instead of an indiscriminate prefix index.

## Measurement log

Run `quickfeed_index_optimization.sql` in MySQL and copy the values from the two `information_schema.TABLES` result sets below.

| Measurement | Data (MB) | Index (MB) | Total (MB) |
|---|---:|---:|---:|
| Before optimization | [record result] | [record result] | [record result] |
| After optimization | [record result] | [record result] | [record result] |

The exact reduction depends on row count, value distribution, InnoDB page allocation, and server version; no fabricated benchmark values are used here. On a populated production table, record the actual `INDEX_LENGTH` reduction.

## Trade-off and limitation

Removing indexes can make some selective reads slower. The correct decision depends on query workload, cardinality, selectivity, and measured plans. For an append-only archive, extra indexes may be acceptable because write maintenance is rare, but they still consume disk and affect backup and cache requirements. `INDEX_LENGTH` is an allocated storage statistic and may not immediately shrink at the filesystem level after dropping an index; verify database-reported value and operational disk usage separately.
