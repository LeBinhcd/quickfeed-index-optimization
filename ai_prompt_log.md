# AI prompt log

## Prompt 1 — Storage statistics

**Question:** How can MySQL report the data and index size of `quickfeed_db.Posts` in megabytes?

**Key result:** Query `information_schema.TABLES`, use `DATA_LENGTH` for table data and `INDEX_LENGTH` for indexes, and divide bytes by `1024 * 1024`. `SHOW TABLE STATUS LIKE 'Posts'` is also useful for a quick inspection.

## Prompt 2 — Cardinality

**Question:** Why are Boolean and three-value columns often poor standalone B-Tree index candidates?

**Key result:** Cardinality is the number of distinct values. A low-cardinality predicate matches a large part of the table, so scanning the table can be cheaper than following an index and fetching many rows. The optimizer makes this decision from statistics and query context; low cardinality does not make an index universally invalid.

## Prompt 3 — Write overhead

**Question:** What happens during an insert when many secondary indexes exist?

**Key result:** InnoDB stores and maintains each secondary index separately. Inserts can require additional page writes, page splits, buffer-pool space, and redo logging. This increases write latency and storage overhead.

## Prompt 4 — TEXT search

**Question:** What should be evaluated for keyword search inside `content`?

**Key result:** A deliberately designed MySQL `FULLTEXT` index may be appropriate for natural-language or boolean text search. It is not automatically a replacement for every query, so search behavior and workload must be tested.

## Prompt 5 — Scope and safety

**Question:** Which indexes should be removed in this exercise?

**Key result:** Based on the supplied decision matrix, remove only `idx_content`, `idx_post_type`, and `idx_is_visible`. Keep the primary key, `idx_user_id`, and `idx_created_at`. No command that blindly deletes every index was used.
