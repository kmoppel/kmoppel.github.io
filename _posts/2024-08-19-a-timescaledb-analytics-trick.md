---
layout: post
title: A TimescaleDB analytics trick
cover-img: /assets/img/timescaledb-analytics-trick.png
tags: [postgres, timescaledb, analytics, performance]
---

While everyone agrees that Postgres has a lot of really nice things going for it, **the analytics story has lagged a bit**
as there are some hard problems to be solved - the on-disk format would have to be changed or IO-layer abstracted away
even further to allow alternative engines, the row based executor etc...

But luckily Postgres is extensible as hell - and taking advantage of that, another cool news items popped up this week -
[native-ish integration](https://motherduck.com/blog/pg_duckdb-postgresql-extension-for-duckdb-motherduck/) with **DuckDB!
Really exciting stuff**, lot of fuzz about DuckDB recently. And hope to give the extension a spin as soon as they
release the binaries - didn't feel like messing around myself. The ensuing [Hacker News discussion](https://news.ycombinator.com/item?id=41275751) also
again highlighted the fact that Postgres’ analytics story could be better…and this prompted me to think back to some
project’s where I’ve battled with the same - trying to make Postgres work while already also considering alternatives.
Mostly resulting in Postgres “winning” though / becoming a "nail"…which is kind of great actually,
as bringing in another piece of tech always carries some non-obvious risks along the line.

So here just a few ideas I've successfully employed in the past where common Postgres patterns were slightly not cutting
it for analytics / Big Data use cases.

# Some ideas to “hack” Postgres for analytics

## Careful data modeling

The obvious first of course - much can be done with pre-sorting, pre-aggregation, non-precise calculations / algorithms where possible,
just the right columns count, correct data-types, liberal use of arrays, app-level caching or caching into materialized views,
roll forward row corrections (e.g. writing a new balance correction entry instead of changing old pre-aggregated stuff) etc etc…

## TimescaleDB with compression

[TimescaleDB](https://www.timescale.com/) in the analytics context might sound surprising to many…as it is, or at least was I think, mainly advertised
as a time-series database…but when it comes to large scale analytics in most cases we’re disk-bound - meaning anything
that decreases the disk footprint is good. And voila - **Timescale has much better compression than Postgres** (with narrow
columns and no TOAST going on there’s basically none) and can shrink the data size ~10x usually, plus has a few other
optimizations.

Here note that while most transactional or financial data is timestamped anyways, **Timescale can be "forced to fit" even if there’s no natural
timestamp** - in many cases it makes sense to invent an artificial timestamp as crazy as it sounds! I’ll showcase a little
example below so that the concept doesn’t stay too abstract…

## Compressed ZFS mounts / tablespaces

In short - ZFS is great also for Postgres! Given it’s mostly applied for heavy reading…I know many people
consider it experimental, are worried about licensing or just haven’t tried it out for some reason - but really, haven’t
had any issues myself over the years.

My general recommendation here would be though to use ZFS only for dedicated minimally-mutating databases or as some
“special” ZFS tablespaces in some more transactional instances.

## File data wrapper + compression for static data

For the fun of it I’ll throw in even the “file_fdw” + “program” option I haven't used recently though, something [a la](https://wiki.postgresql.org/wiki/New_in_postgres_10#file_fdw_can_execute_a_program)

```sql
   CREATE FOREIGN TABLE
   test(a int, b text)
   SERVER csv
   OPTIONS (program 'gunzip -c /tmp/data.czv.gz');
```

Note that this only makes sense for very large + static / non-transactional datasets in conjunction with very slow disks
(that are increasingly becoming rare) that otherwise would cause way too much disk reading.

Also make sure you don't use gzip as in the example, but something better (lz4, zstd) and have
super narrow tables, basically going plain-file based columnar! For ultra large datasets you might want to look at
[parquet_fdw](https://github.com/adjust/parquet_fdw) though.


# An example TimescaleDB vs vanilla Postgres win

To give an impression of what could be the benefit of TimescaleDB compression I also stitched together a small analytics
test case based on the pgbench data model, using relatively slow disks + little RAM to highlight the gains.

**Hardware**: AWS EC2 M1 General Purpose Large
m1.large  7.5 GiB 2 vCPUs 840 GB (2 * 420 GB HDD)

**Postgres**: 16.4 + Postgres config tuned with [timescaledb-tune](https://docs.timescale.com/self-hosted/latest/configuration/timescaledb-tune/)
for the given hardware 

**Dataset**: 200B rows of pgbench_accounts data with an extra synthetic timestamp column, needed for TimescaleDB chunking,
26GB (pre-compression) i.e. 4x RAM, meaning pretty small

**Query**: Calculating the average account balance per branch i.e. a full dataset GROUP BY -
```select bid, avg(abalance) from pgbench_accounts group by bid```

**Results**:

Avg. runtime vanilla **Postgres: 239s**

Avg. runtime compressed **TimescaleDB: 23s**

Full test script [here](https://gist.github.com/kmoppel/3f5ad9101cf15ad0678482460ad650db#file-timescale_vs_native_analytics-sh)

As we see - an easy 10x win here…I wouldn’t expect that much for most use cases though as synthetic data compresses too well…
but still..not bad - given that we don’t lose on the OLTP side also when recent chunks are not compressed.

Btw, another bonus point for TimescaleDB - many don’t know that it has a huge amount of analytical [functions](https://docs.timescale.com/use-timescale/latest/hyperfunctions/about-hyperfunctions/)
that provide stuff not existing in Postgres or which is just faster than common workarounds!


# Summary

In many cases Postgres for analytics bogs down due to too much data / slow disk access - then some simple band-aid can work
(with CPU-bounded workloads it’s much tougher though). A separate question of course is if such approaches could be 
considered cool or modern…feel free to voice your opinion below in the comments :)

In short - just wanted to refresh your memory that “tech things™”, including Postgres, are for the most part built like 
onions - and one can mess around with the layers a bit…given you know what you’re doing of course :)

Also on that note - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a hand with Postgres - I’ve
been working with Postgres daily for the last 10+ years with some of the most Postgres-heavy companies in Europe.
