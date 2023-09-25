---
layout: post
title: Postgres 16 - a quick perf check
tags: [postgres, performance, pgbench]
---

“That time of the year”™ again 🎉

As per usual for that "season" - not that I don’t trust the PostgreSQL Global Development Group and its performance farm (they did catch
some performance regressions in beta stages!) - but somehow as soon as a new major release arrives, I get some weird urge
to personally put it on the testbed 🤷 One's own eye is still the king I guess...

So I booted up my two old servers collecting dust in the corner (as “jitter” for cloud instances is still quite huge and it
masks most *$ver vs $ver+1* changes) and started a script which I have around since a year ago (with some light modifications).
So here a quick shout-out to the world also on my key "findings".

*TLDR;* - The most recent version of Postgres seems “business as usual” again - very small performance upticks based on
a quick mostly OLTP-style check, one could say possibly even the "most boring" Postgres release in recent times...which is generally good of course I guess:)

## Test setup

**Hardware**: 2x on-prem (under my desk) old workstations, 4 CPU (Intel i5 and Xeon E3 both @ 3.30GHz, no hyperthreading), 16GB RAM, SATA SSD

**OS**: Ubuntu 22.04 Server

**Postgres**: latest v15-series (15.4) vs v16 from the official [PGDG](https://wiki.postgresql.org/wiki/Apt) repos

**Working set size**: *pgbench* in-mem vs light disk access (3x RAM), no partitions vs 16 vs 128 range partitions

**Measuring method**: Postgres built-in "pg_stat_statements" extension stats aggregation

**Level of parallelism**: Low, client count = $CPUs for most test queries, which almost fully utilizes the CPU though on local testing

**Test queries**:
  - Key reads (*pgbench \-\-select\-only)*
  - Batched key reads (*aid* BETWEEN random + 10k)
  - Key updates *pgbench \-\-skip\-some\-updates* (does also one extra SELECT and INSERT)   
  - Batched key updates (*aid* BETWEEN random + 10k)  
  - Full scan (*SELECT max(abalance) FROM pgbench_accounts*)

**Test duration**: 4h per scale / partitions count / query / pgver, total of ~10 days

**Postgres config**: [Pgtune](https://pgtune.leopard.in.ua/) OLTP suggestions for my HW + a few custom changes:

```bash
max_parallel_workers_per_gather = 1 # Optimal for my 4 CPUs
unix_socket_directories = '/tmp' # To run under any OS user conveniently
shared_preload_libraries = 'pg_stat_statements'
wal_compression = zstd # Should be the default nowadays for v15+
track_io_timing = on # To get IO call duration in pg_stat_statements
synchronous_commit = off # Don't want to test disk COMMIT throughput
```

The full test script can be found [here](https://github.com/kmoppel/pg-perf-test-v10-v15/tree/v15-vs-16-pgbench)

PS If you plan to run your own test with built-in *pgbench* test for comparison, note that in addition to the standard
*pgbench* schema, I added an index to the *pgbench_accounts* table to enable hypothetical quick "top X accounts for branch Y"
queries (to try to sell them crypto investment products :money_mouth_face:) as the default only-one-index pgbench_accounts schema
is a very unrealistic use case for OLTP, catering for too many HOT updates.
	
## Results 

So fast-forward 10 days - I started to eagerly check the results...and after double-checking the data I was left a bit 
disappointed - they were literally the most uninteresting test results I've ever gotten, not much change at all! We're talking
about a few percentages here and there for most query + scale + partitions configurations, and **a total of ~1.2% overall
improvement** over all 6 different queries. Interestingly the total standard deviation on the other hand increased by a percentage,
but as most queries executed in ~1ms, I guess could be normal measuring variance. 
For a total runtime of 20 days (10d on 2 nodes) I'd say this is some pretty impressive stability still in any case!

So yeah, this time no point even to paste the results table here, too boring, will save you some time :)

But please do run the test yourselves if you have time (full test script link above) and reach out to me if it looks way
different for you - I'm still actually a bit suspicious of the fact that the differences were so small...

PS To be honest - out of all the combinations one (in-memory key reads with no partitions), was also slower above
the usual few percentages (~10%) - but didn't have much influence on the big picture (20 permutations) and I think could
be discarded in the end as this is rather uncommon use case in hindsight - databases usually outgrow memory heavily and
for in-memory key-value you'd be better off with Redis or such.



## Key learnings

As already pointed out - **"sadly" not much to say this time**:

* Postgres has become so stable seems, that it has become very hard to squeeze out anything from the most common OLTP queries
* v16 did a tiny bit better in "key-range access" execution times (~7%)
* v16 did a tiny bit worse in "in-memory random key reads with no partitions" execution times (~10%)
* Partitioning (0 vs 16 vs 128 was tested) did influence the query runtimes (~12%) in a negative way for this disk-light testing.
  In regards to v16 though, it again performed minimally (around ~2%) better with partitions.
  - The reason for this degradation seems to be lower Shared Buffer hit rates - not sure what's the root cause there though.

And as testing is a tricky exercise, some departing words - note that:

* It's an artificial testset with still a pretty short runtime of 4h per pgver / query / scale / partitions combination
* Hardware and dataset-to-memory ratio are as important as ever and one should try to test close to their data size and use
  close-to-domain tables and queries - this test didn't touch disk much
* These six tested queries represent a fraction of typical SQL constructs + amount of rows touched per query
