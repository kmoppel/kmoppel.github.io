---
layout: post
title: The bountiful world of Postgres indexing options
tags: [postgres, indexing, btree, hash, pgbench]
---

Last week’s Postgres Weekly had quite an interesting [piece](https://sirupsen.com/index-merges) that compared the performance
of composite indexes vs normal indexes being merged on-the-fly. And especially interesting - throwing MySQL also into the
ring! Recently "sadly" not too much on that comparison front somehow - ah, those good old flame war days...

Anyways - although quite some necessary testing details  (exact engine versions, hardware, dataset disk / memory ratio,
any config tunings) for a complete performance overview were left out - I think the rule of thumb derived in the article
basically is valid and it’s a nice and simple thing to remember - **“10x”** for the win ! But he also left out a couple of additional
index types that could be considered on Postgres side, given some assumptions stand. Can’t really blame him though -
as indeed there are quite some Postgres indexing strategies and index types available that don’t pop up too often! So here
are two more alternative (thus not necessarily better) approaches from my side that came to mind when looking at that piece,
together with some benchmarking numbers.


## Prerequisite - generating test data

Sadly the article didn’t provide a test data generation script so needed to guess a bit -  especially the correlation factor
could be actually very influential when designing indexes, as in ideal case one could then also throw BRIN into the bunch,
but I’m assuming worst case here - random data distribution.



```sql
CREATE UNLOGGED TABLE test_table (
  id bigint /*primary key*/ not null,

  text1 text not null, /* 1 KiB of random data */
  text2 text not null, /* 255 bytes of random data */

  /* cardinality columns */
  int1000 bigint not null, /* ranges 0..999, cardinality: 1000 */
  int100 bigint not null, /* 0..99, card: 100 */
  int10 bigint not null /* 0..10, card: 10 */
);

INSERT INTO test_table
SELECT
  id,
  (select string_agg(random()::text,'') from generate_series(1,52)) text1, /* length(random()::text) ~19B */
  (select string_agg(random()::text,'') from generate_series(1,14)) text2,
  random()*1000 int1000,
  random()*100 int100,
  random()*10
FROM
  generate_series(1, 1e7) id; /* 10M rows ~ 13GB table */

VACUUM ANALYZE test_table;

SELECT pg_size_pretty(pg_table_size('test_table'));
 pg_size_pretty 
────────────────
 13 GB
(1 row)
```
	
## Starting point - composite and merged indexes

To set a baseline for comparisons - the first tests I ran mimicked the ones from the referenced [article](https://sirupsen.com/index-merges).
In addition I wanted to blend out randomness of a few individual calls though and made it look like more classical benchmarking
using “pgbench”, running each type for half an hour on random input.

Results are summarized at the end of the article.

```bash
SQL=$(cat << "EOF"
\set int1000 random(0, 999)
\set int100 random(0, 99)
SELECT count(*) FROM test_table WHERE int1000 = :int1000 AND int100 = :int100;
EOF
)

psql -Xc "CREATE INDEX test_table_composite ON test_table (int1000, int100);" postgres

echo "$SQL" | pgbench -h /var/run/postgresql -n -f- -T 1800 -c 2 postgres

psql -Xc "DROP INDEX test_table_composite;"
psql -Xc "CREATE INDEX test_table_int1000 ON test_table (int1000);"
psql -Xc "CREATE INDEX test_table_int100 ON test_table (int100);"

echo "$SQL" | pgbench -h /var/run/postgresql -n -f- -T 1800 -c 2 postgres
```

PS Note that we’re running here without the EXPLAIN as it adds some overhead usually not wanted for benchmarking.
Also for a more scientific test one should further use “pg_stat_statements” for reading the timing information, but for a
simple comparative test I think we’re fine here.

## Meet covering indexes - cheap composite index alternatives

[Covering a.k.a. Including indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html) have been around
for 4 years already since v11, but still pretty underused - and maybe even with a reason. Namely they have a few downsides -
mainly that the “payload” components can’t be used for ordering and getting fast filtering on payload columns (in addition
to the required main columns) depends on index-only scans - i.e. one needs to have relatively few writes / updates and long-running
transactions or more aggressive Autovacuum settings to keep the Visibility Map efficient.

Creating these indexes can be identified by the INCLUDE syntax:

```sql
CREATE INDEX test_table_covering ON test_table (int1000) INCLUDE (int100);
```

NB! Make sure to always put the most selective column first! Supports also multi-column in the first, “normal” index part.

Execution plan:

```
EXPLAIN SELECT count(*) FROM test_table_old WHERE int1000 = 100 AND int100 = 10 ;
                                                      QUERY PLAN                                                       
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Aggregate  (cost=243.82..243.83 rows=1 width=8)
   ->  Index Only Scan using test_table_int1000_int100_idx on test_table (cost=0.43..243.56 rows=107 width=0)
         Index Cond: (int1000 = 100)
         Filter: (int100 = 10)
(4 rows)
```

## Another rarity - Hash index merge

Hash indexes are actually first class (WAL logged i.e. crash-safe) Postgres citizens for a few years already, but still
pretty much fly under the radar. Probably rightfully again so, as this fella is basically a one-trick pony! :horse: Albeit an
effective one, when you stumble upon the particular use case of quickly locating a needle (key) from a multi-Terabyte
haystack - then it kicks B-tree’s ass and provides O(1) access times. But again - no range search, no unique indexes,
no multi-column, considerable disk footprint - so generally not worth it.

Creation:

```sql
CREATE INDEX test_table_hash_int1000 ON test_table USING hash (int1000);
CREATE INDEX test_table_hash_int100 ON test_table USING hash(int100);
```

Execution plan:

```
EXPLAIN SELECT count(*) FROM test_table WHERE int1000 = 100 AND int100 = 10 ;
                                             QUERY PLAN                                              
─────────────────────────────────────────────────────────────────────────────────────────────────────
 Aggregate  (cost=17.35..17.36 rows=1 width=8)
   ->  Bitmap Heap Scan on test_table  (cost=16.23..17.35 rows=1 width=0)
         Recheck Cond: ((int1000 = 100) AND (int100 = 10))
         ->  BitmapAnd  (cost=16.23..16.23 rows=1 width=0)
               ->  Bitmap Index Scan on test_table_hash_int1000  (cost=0.00..1.88 rows=104 width=0)
                     Index Cond: (int1000 = 100)
               ->  Bitmap Index Scan on test_table_hash_int100  (cost=0.00..14.10 rows=1000 width=0)
                     Index Cond: (int100 = 10)
(8 rows)
```

## Query runtimes summary

Query:
```sql
SELECT count(*) FROM test_table WHERE int1000 = :int1000 AND int100 = :int100;
```

Results as reported by "pgbench":

| Index type   | Avg. runtime for 10M rows (ms) | Index size (MB) |
|--------------| ------------------------------ | --------------- |
| Composite    | 0.06                           | 69              |
| B-tree merge | 20.3                           | 133 (66+67)     |
| Covering     | 0.89                           | 301             |
| Hash merge   | 22.1                           | 892 (448 + 444) |

Test server: 4 CPU (Xeon E3-1225 v6 @ 3.30GHz), 16GB RAM, SSD, Ubuntu Server 22.04, Postgres v15.1, config all at defaults except “random_page_cost=1.25”

The full executable test script can be found [here](https://gist.github.com/kmoppel/9ba1f938140448dc1919a21c215196b1#file-full_index_test_10m_rows-sh),
if you want to re-run on your own hardware.

## Learnings / thoughts

* All index methods basically provided acceptable OLTP response times for our smallish 10M rows (in memory) test case.
* Covering indexes also provide a very good response time, and it might make sense for older Postgres versions (v11, v12)
  where index de-duplication was not available - otherwise the larger size makes them much less attractive. They might be
  faster for heavy UPDATE use cases though as in theory the structure is simpler - no sorting needed (didn’t look into it though).
* The testset was too small to show benefits of hash indexes (and probably also the cardinality too low) - the dataset size
  should have been probably 100x or more or just should have tested on much slower disks. Hope to test that in the future,
  but still surprisingly close to normal B-tree merge so there might be some potential there - as remember - hash means O(1)
  and at some point (>TB) the huge index size penalty should amortize.
* We disregarded data modifications here - in reality this definitely would have to be factored into the “winning formula”
  as some index types tend to bloat more and then also the Visibility Map / Autovacuum lag steps in and index-only scans
  start to lose efficiency.
* If one can swap out “int8” for normal “int” for the indexed / queried data columns, one could also try out Bloom
  indexing - the more columns you want to “merge” together, the more sense it should make and the performance benefits
  can be considerable - as I once found out [here](https://www.cybertec-postgresql.com/en/trying-out-postgres-bloom-indexes/).
