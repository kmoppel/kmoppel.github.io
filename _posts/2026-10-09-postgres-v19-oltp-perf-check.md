---
layout: post
title: An OLTP perf check on Postgres 19 Beta 3.5
cover-img: /assets/img/v18_v19_race_2.jpeg
tags: [postgres, performance, pgbench, tpcc]
---

Yay, “That time of the year”™ again soon 🎉

This time maybe with a bit more fuzz / excitement / anxiety as usual - due to various obstacles surrounding the release as detailed in a few recent posts by various people.
Not going into the "Is AI to blame or not" here - **the fact is that more code has changed than usual** (numerically for example
`git diff --shortstat` gives us +15% in files changed, +38% insertions, +75% deletions compared to v18) - and due to the revert spike
there's a bit less of that usual *it's gonna be sweet as usual* gut feeling.

To regain some of that gut feeling I decided to check if the main bread of Postgres so to say - OLTP / System of Record
side of things - is not negatively affected. To do that I threw a few hundred bucks of my own money at clouds, **running 2 OLTP workloads on
a variety of different hardware on latest stable v18 and v19 Beta 3 and Beta 4** - roughly with a 50-50 split, yielding a
tongue-in-cheek virtual version number "3.5" mentioned in the title. 

**TLDR;** You shouldn't worry too much about PostgreSQL's bread-and-butter OLTP capabilities with release 19! The opposite -
my (maybe somewhat simplistic) OLTP testing (~800 hours / 33 days of total runtime nevertheless) found a small uptick of a few percentage points
in *pgbench* and *TPCC-like* handling - which should at least hint that there won't be any disasters in that section :)


# Test setup

**Postgres**: v18.6 vs v19 Beta 3/4 from the official [PGDG](https://wiki.postgresql.org/wiki/Apt) repos

**Hardware**: 4-96 vCPU (m7gd.xlarge, m8id.xlarge, c5d.2xlarge, c7gd.4xlarge, c5d.metal), 16-192 GB RAM, SSD / NVMe storage (~2/3 run on AWS EC2, 1/3 on Hetzner)

**OS**: Ubuntu 26.04 Server and Debian 13

**Working set size**: In-mem or some light disk access only

**Measuring method**: Postgres built-in "pg_stat_statements" for individual queries + test durations comparisons. 

**Level of parallelism**: Client count depending on CPU count, but always smaller than the CPU count with the aim of not
to test the kernel scheduler instead.

**Tested workloads**:
  - Key reads (pgbench *\-\-select\-only)*
  - Batched key reads (pgbench *aid BETWEEN $random + 10k*)
  - Key updates *pgbench \-\-skip\-some\-updates*    
  - My pgbench-optimized [flavour](https://github.com/kmoppel/pgbench-tpcc-like) of the TPCC test

**Test duration**: 1-10m transactions per client / configuration / loop.

*PS* Last time I ran a similar test I used time-boxing (a la 4h for each permutation of variables) - but since that I've
learned that using a constant amount of work, i.e. a fixed transaction count is better / more equal. 

**Postgres config**: Minimal "best-practice" touches for given HW + a few custom changes. See for example [here](https://github.com/kmoppel/pg-perf-test-v10-v15/blob/v18-vs-v19/postgresql.tune.16-cpu.conf) for a 16 vCPU config.

```bash
# shared_buffers, max_connections, work_mem, maintenance_work_mem ~ f(vCPU)
random_page_cost = 1.25
effective_io_concurrency = 200
wal_compression = on
track_io_timing = on
jit = off # as off in v19 by default
```

Also note that Autovacuum was disabled dynamically by the test scripts! And a fillfactor of 80% was used for pgbench init,
aiming to simulate a near to real-life situation where we've been running for a while already and have "holes" in the heap.

In hindsight I probably should have increased the `io_workers` for the larger instances as well...but probably OK-ish,
given scale / DB size was chosen to be in-mem or near that.

**Other variables**

For some runs I also varied:

 * the partition count
 * query protocol (simple / prepared)
 * sync vs async commit (via `synchronous_commit`, i.e. ignoring the actual wait for WAL flushing on COMMIT)
 * setting a fixed pgbench random seed vs a default time-based one

**Test runner scripts**

The full test scripts can be found here:

* The [pgbench runner](https://github.com/kmoppel/pg-perf-test-v10-v15/tree/v18-vs-v19)
* [TPCC-like runner](https://github.com/kmoppel/pgbench-tpcc-like-benchmark)

PS Note that the *pgbench* schema differs a tiny bit from to the default layout as I've added an index to the *pgbench_accounts.bid*
column to try to look a tiny bit more real-life.
	

# Results 

Looking at the numbers after a few weeks of test running (SQL to analyze the stats pushed to the $resultsDB are in the
repos if you decide to run your own testing) I could smile relaxedly - things in the OLTP section seem stable and slowly improving!

If to put a "magic" number on it (please note the nuances from below though!) - over both test-sets and all variables (in-mem vs light disk,
partitions vs no partition, query protocol, sync vs async) **a total of ~3% test exec duration improvement** could be witnessed!

**BUT**...if to drill into individual queries / test modes - the picture gets sadly a bit more fuzzy. For example the total
average improvement over various SQL statements types (SELECT, INSERT, UPDATE) was actually ~7%...but runtime only improved by 3%.
Where did the difference go? Well...I guess *pg_stat_statement* doesn't show the whole session / transaction and lock management
story still.

With some certainty (quite some hours were churned to blend out the noise) though I think I can say:

* SELECT-s (both key and batch) were a bit faster - as measured by *pg_stat_statements*.
* pgbench INSERT-s were consistently A LOT (~25-30%) faster! This was the only real surprise for me in this testing - should actually dig deeper here to see from where it's coming from...
* pgbench key UPDATE-s were same or a tad slower on v19 on average.
* The TPCC-like workload (more realistic - more tables / indexes) gained a bit more performance compared to pgbench "tpcb-like".


## The "jitter" disclaimer

As anyone who has done benchmarking on the clouds knows:

 - Correct benchmarking is generally hard! See for example a pair of articles from Tomas Vondra [here](https://vondra.me/posts/stabilizing-benchmarks/)
   and [here](https://vondra.me/posts/benchmarking-is-hard-sometimes/) and a [YouTube](https://www.youtube.com/watch?v=8Y6GfTY5eFE) from Andres Freund
   to get an idea on keywords like: warmup, OS caches, NUMA, Huge Pages, schedulers, CPU power states, kernels, ... 
 - There's a healthy amount of jitter involved - short runs don't show much.

This time I think I saw a lot more jitter compared to my last similar testing. Had to even blend out some clear outliers - crazy stuff seems can happen
on standard AWS EC2 VM's seems - prompting me to eventually rent a relatively expensive "metal" instance (I used Spot VM-s
of course via my [pg-spot-operator](https://github.com/pg-spot-ops/pg-spot-operator) utility) and throw a different provider (Hetzner) in the mix as well.

But please do run the tests yourselves if you have time (links above) and reach out to me if the results look way different for you.

Also keep in mind that these 2 workload represent a fraction of the real world and Postgres versions are best tested following
as precisely your business schemas and access / write patterns as possible.


# A note on the Postgres Performance Farm project

By the way - wouldn't it be nice if one could open some link and get some similar numbers shown without much ado for some
common workloads / benchmarks for each Beta release? And this was, once, the aim for a now-stale Postgres-roofed community project called [PGPerfFarm](https://github.com/PGPerfFarm)
as well...would be really-really nice to resurrect it I think! I could do my part as well - feel free to ping me if the project is still
on someone's mind, or there's already some better equivalent that I haven't heard about yet. Thank you!
