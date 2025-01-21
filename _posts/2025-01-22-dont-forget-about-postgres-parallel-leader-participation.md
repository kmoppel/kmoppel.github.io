---
layout: post
title: Don't forget about the Postgres parallel leader participation setting
cover-img: /assets/img/parallel_leader_participation.jpeg
thumbnail-img: /assets/img/parallel_leader_participation_thumb.jpeg
tags: [postgres, bigdata, parallelism, testing, pgbench, ansible]
---

Recently there was a nice [article](https://www.crunchydata.com/blog/parallel-queries-in-postgres) on the Planet PostgreSQL
feed  on Postgres’ parallel query capabilities and the pertinent tuning parameters. All good and logical, the main settings
and considerations were highlighted…but then I suddenly remembered that there’s one more parallelism relevant parameter that for some
reason is mostly always overlooked - ["parallel_leader_participation"](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-PARALLEL-LEADER-PARTICIPATION).
Has this poor parameter lived in the shadows for some good reason? No idea. Time for an experiment…

**TLDR;** You should actually look into disabling *parallel_leader_participation* if you’re working with larger out-of-cache datasets,
have plenty of cores and especially when the tables are partitioned.

# parallel_leader_participation - a known unknown

Postgres [docs](https://www.postgresql.org/docs/current/runtime-config-resource.html#GUC-PARALLEL-LEADER-PARTICIPATION)
state what it does on high level of course:

```
Allows the leader process to execute the query plan under Gather and Gather Merge nodes
instead of waiting for worker processes. The default is on. Setting this value to off
reduces the likelihood that workers will become blocked because the leader is not reading
tuples fast enough, but requires the leader process to wait for worker processes to start
up before the first tuples can be produced. The degree to which the leader can help or
hinder performance depends on the plan type, number of workers and query duration.
```

which though in general leaves an **“it depends”** impression - so let's try to test at least once common use case - full
table scan analytics.

# Test setup

Obviously to parallel settings start to matter only when we have quite some CPUs and data at our hands - so needed to
ditch my beloved old 4 core test server collecting dust under the table, and look towards the cloud this time. To keep
costs under control I used 6 AWS Spot VMs, which by the way are effortless to provision using my [PG Spot Operator](https://github.com/pg-spot-ops/pg-spot-operator)
utility. 


* **CPU:** 16 vCPU
* **RAM:** 32 GB
* **Storage:** c7g.4xlarge and c6g.4xlarge with EBS gp3 volumes (3k IOPS, throughput increased to 500 MBs), 3x c6gd.4xlarge with 950GB of fast local storage
* **Dataset:** pgbench in-memory (scale=2000, 25GB) and 2x RAM (scale 5000, 63 GB)
* **Partitioning:** pgbench \-\-partitions=0\|8
* **Query:** Full scan aggregate: `select bid, avg(abalance) from pgbench_accounts group by bid`
* **Parallelism:** max_parallel_workers_per_gather=2\|4\|8\|16
* **Runtime:** 8h per VM, 48h total (30min per scale / partitions / max_parallel_workers_per_gather / parallel_leader_participation combination)
* **Measuring:** pg_stat_statements mean_exec_time

Full test scripts here: [https://github.com/kmoppel/parallel-leader-participation-test](https://github.com/kmoppel/parallel-leader-participation-test)

# Results

Let’s let the summary numbers (SQL-s for analysing the results [here](https://github.com/kmoppel/parallel-leader-participation-test/blob/main/analyze_results.sql)) speak for themselves:

|scale|partitions|mean_exec_time_s| no_leader_participation_speedup_pct |
|-----|----------|----------------|-------------------------------------|
|2000 |0         |5               | 18.6                                |
|2000 |8         |6               | 20.0                                |
|2000 |NULL      |6               | 19.3                                |
|5000 |0         |362             | -0.6                                |
|5000 |8         |308             | -6.1                                |
|5000 |NULL      |335             | -3.3                                |
|NULL |NULL      |170             | 8.0                                 |

Hmm, a mixed bag indeed as the documentation hints...on average it slows things down for our use case. But if to try conclude
something - **the target use case for disabling “parallel_leader_participation” seems to be larger-than-cache
working sets + partitioned tables**. In other cases we actually lose or are on the border of noise though, so some caution
is required.

Also - when looking at more detailed results one could see that the improvement was
slightly bigger for network disks with slower latencies compared to local SSD-s, so this might be an additional "target
use case" factor.

# Summary

There’s a reason why the “parallel_leader_participation” parameter is not enabled by default - it **can actually make things
slower!** As we saw with the in-memory (<10ms response times) aggregation results...

But - when dealing with **out-of-RAM datasets** and machines with **higher *max_workers_per_gather* settings** (>8), and
especially when **using partitions** - one could easily get a **small boost**! With up to 11% observed on 1 out of 6 test VMs.
And mind you - I only tested with a relatively small, half-cached dataset.

In the end not game-changing of course, but I think enough to give *parallel_leader_participation* an appreciative pat on the back
now and then. **I would not recommend to enable it globally** still though, but rather per query or session for heavier analytics queries.


*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
