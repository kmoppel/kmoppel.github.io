---
layout: post
title: Can there be too many partitions?
tags: [postgres, performance, pgbench]
---

Quite some time ago at work there was a need to partition a particularly huge table, because: *a)* bloat was getting out of
hand - as, remember, so far only one Autovacuum worker can work on a single table or subpartition *b)* there might arise a
need to extract / move some busy partitions to separate instances, and access either over *postgres_fdw* or make the app
"shard" aware. 

Ok all good and doable (well, still needed a bit of chemistry like background triggers to re-write the old data in the
background and double-write the incoming rows to the new table at the same time as it had to be a live migration) …but hmm, wait, what is
actually a good  number of partitions? Is there a danger of hurting yourself or the database if you have too many?
Will planning and execution times change significantly when you cross some sort of a threshold?

As there seemed to be little conclusive evidence on the interwebs on that, I decided to "roll my own" quick test, as I
usually do in such cases :) But sadly never at the time actually got to the part where I jot something down about it.
Still - it stayed at the back of my head for some reason - I guess because pretty hard to avoid “big data” and partitioning nowadays...
And as an extra annoyance - the Postgres version I used at the time had in the meantime aged horribly, so I had to re-run the
test scripts with the latest Postgres version (v15.2). So read on for a few numbers and some conclusions on my test case
with RANGE and HASH partitioning and partition counts going from 0 to 4096.

**TLDR;** - Postgres handled increasing partition counts very well for a simplistic "SELECT * FROM tbl WHERE id = $1" use case
and "sadly" nothing to complain or worry about! :smirk:


## Test setup

**Hardware**: 2x on-prem (under my desk) old workstations, 4 CPU (Intel i5 and Xeon E3 both @ 3.30GHz, no hyperthreading), 16GB RAM, SATA SSD

**OS**: Ubuntu 20.04 Server, set to full CPU performance mode

**Postgres**: v15.2 from the official [PGDG](https://wiki.postgresql.org/wiki/Apt) repos

**Working set size**: pgbench scale 5000 ~ 73 GB DB size, i.e. an active set of ~5x RAM

**Measuring method**: Postgres built-in "pg_stat_statements" extension 

**Level of parallelism**: Low parallelism / load to decrease randomness as measuring smallish time intervals

**Pgbench test mode**: *\-\-select\-only* in "simple" protocol mode, i.e. no plan caching to see the cost of planning. Random and Zipfian "Top 5%" access patterns.

**Test duration**: 1h for each partition method, partition count and access pattern on 2 servers, 56h in total

**Postgres config**: All defaults except:

```bash
shared_preload_libraries='pg_stat_statements'
pg_stat_statements.track_planning=on # To also measure planning times
shared_buffers=4GB # Common starting point of 1/4th of RAM
```

Full test script can be found [here](https://gist.github.com/kmoppel/b42704e4115e04bf1947be8007788f46) by the way.

## Results

One could look at many things I guess, but for me of most interest was to see how much planning and execution penalty does increasing
the partition count carry (compared to no partitions at all) in case of both partitioning methods, to see if should
try to prefer one. Although in practice I guess most of the time it's actually almost impossible to go with both methods for a particular use case - some data
fits better the RANGE model, some data HASH. And some more rare cases even the LIST model - which generally involves a
too much "micromanagement" though.

**Effect of partition increases for HASH partitioning**

| Row access pattern | Partitions | Mean plan time change (%, compared to no partitions) | Mean exec time change (%) |
|----------------|------------|------------------------------------------------------|-------------------------|
| random_access  | 0          | 0                                                    | 0                       |
| random_access  | 16         | 51.8                                                 | 4.6                     |
| random_access  | 64         | 58.2                                                 | 5.6                     |
| random_access  | 256        | 67                                                   | 3.2                     |
| random_access  | 1024       | 107.2                                                | -3.5                    |
| random_access  | 4096       | 128.8                                                | -5                      |
| zipfian_access | 0          | 0                                                    | 0                       |
| zipfian_access | 16         | 49                                                   | 4                       |
| zipfian_access | 64         | 53.4                                                 | 4.7                     |
| zipfian_access | 256        | 60.6                                                 | 1.6                     |
| zipfian_access | 1024       | 104.7                                                | 0.9                     |
| zipfian_access | 4096       | 125.3                                                | -0.9                    |

**Effect of partition increases for RANGE partitioning**

| Row access pattern | Partitions | Mean plan time change (%, compared to no partitions) | Mean exec time change (%) |
|----------------|------------|---------------------|-------------------------|
| random_access  | 0          | 0                   | 0                       |
| random_access  | 16         | 59.1                | 3.2                     |
| random_access  | 64         | 64.9                | 3.2                     |
| random_access  | 256        | 72.9                | 1.7                     |
| random_access  | 1024       | 113.5               | -2                      |
| random_access  | 4096       | 137                 | -6.8                    |
| zipfian_access | 0          | 0                   | 0                       |
| zipfian_access | 16         | 53.8                | 2.7                     |
| zipfian_access | 64         | 55.1                | 2.4                     |
| zipfian_access | 256        | 57.6                | -1.4                    |
| zipfian_access | 1024       | 66.3                | 1.5                     |
| zipfian_access | 4096       | 79.6                | -0.5                    |

Also I wanted to see what is the average penalty when jumping from one "level" of partitions to the next one. Btw, as a "level unit"
for this test setup I chose 4x multiplication just based on my gut feeling, e.g. going from 16 to 64 partitions.

| Row access pattern      | Partition method | Avg plan time (ms) | Avg plan time change (%) | Avg plan time stddev change (%) | Avg exec time | Avg exec time change (%) | Avg exec time stddev change (%) |
|----------------|------------------|--------------------|--------------------------|------------------------------|---------------|--------------------------|---------------------------------|
| random_access  | hash             | 0.036              | 19.2                     | 32.2                         | 0.2482        | -0.9                     | 1.2                             |
| random_access  | range            | 0.0371             | 20.4                     | 36.1                         | 0.2512        | -1.3                     | -0.9                            |
| zipfian_access | hash             | 0.0331             | 18.8                     | 37.1                         | 0.0484        | -0.1                     | -0.1                            |
| zipfian_access | range            | 0.0303             | 14                       | 38                           | 0.0484        | 0                        | 0                               |


PS Full test results as an executable SQL dump file can be found [here](https://gist.github.com/kmoppel/f7c92236e4ece0243d6f39dee76f034b)
and some query ideas to analyze those results [here](https://gist.github.com/kmoppel/b6a93e0bdbaeb5a1cfe8a78327393de3). 

## Key learnings

When later looking at the super-fast execution times I sadly realized that I should have thrown in also some more complex
queries as I guess key lookups are not where Postgres dominates the market exactly. But to make life a bit harder for the
planner I at least threw in another index on the *pgbench_accounts.aid* column. And for the larger end of partitions
(4096) 73GB of data probably also is not a real life match - but, nevertheless I think we can say something:

* Postgres seems to handle a "normal" amount of partitions very well - planning time seems to increase steadily in percentage
  wise, but in real numbers wasn't noteworthy.
  * When partitions counts went from 0 to 4096, avg planning time increased only ~2.5x
  * Increasing the partition count by 4x incurred and approx ~20% penalty in planning times
  * Execution times barely changed for a key based and indexed lookup
* There also seems to be no big difference if using HASH or RANGE partitioning.
  * Maybe the only slight anomaly was that RANGE partitioning worked consistently better with a Zipfian "top 5%" distribution
    than other combinations 
* Going from no partitioning to some reasonable amount increases the execution times by a tiny bit!
  * The difference was always small (a few percentage) in our case, but still a constant factor
  * Seems to boil down to a decreased Shared Buffer hit rate (i.e. a more fragmented cache) as it correlated well with *$partition_count++*
* Standard deviation a.k.a. "jitter" for planning times was much higher than for the actual execution times.
  * Probably due to the fact that we were in the *~30-40 microseconds* territory  
* Be aware that things could look very different when doing cross or intra-partition joins as there seem to be some
  nuances hinted at by the *enable_partitionwise_aggregate* / *enable_partitionwise_join* parameters, which are disabled
  by default not to strain the planner too much.
* First try to bootstrap the schema with 4096 partitions weirdly enough actually failed for HASH partitions! As seems Postgres defaults can't handle dropping
  1K partitions / tables in one transaction - `ERROR:  out of shared memory HINT:  You might need to increase max_locks_per_transaction.` 😿 ...
  * Worked after increasing *max_locks_per_transaction* from default *64* to *128*. Might need another bump for some uncanny partition counts. 
* Note that planning time can, and usually is, largely avoided for repetitive calls via plan caching / prepared statements altogether!
* Also note that if you're not struggling with some planning anomalies (or not doing perf testing :smirk:) then you should leave
  the `pg_stat_statements.track_planning` setting at defaults, i.e. `off`, as it can unnecessarily eat CPU cycles.