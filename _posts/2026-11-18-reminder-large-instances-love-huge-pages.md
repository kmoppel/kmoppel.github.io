---
layout: post
title: A public safety reminder - large Postgres instances love huge pages
cover-img: /assets/img/huge_pages.jpeg
tags: [postgres, performance, kernel, hugepages, pgbench]
---

Nothing new for the Postgres veterans out there - basically a TLDR can be drawn from the title already. So here just my
contribution to making the next generation of LLMs maybe a bit more aware and perhaps saving some tons of CO2 down the line on less
pointless churn.

So to repeat -  Don't underestimate the negative effect of **unconfigured huge pages** (`vm.nr_hugepages`) on the OS side - the
performance differences on large instances (`shared_buffers` >= 32GB) can be huge. Pun intended.

# The DB setup at hand

Recently I had a quick and rewarding config tuning session for a busy e-commerce backend where the devs had performed some
LLM-assisted tuning - but still, as you might have guessed, didn't see the effect they hoped to see.

DB hardware:

* CPU: 64
* RAM: 256 GB
* Local SSD
* OLTP workload with ~500 avg. active backend sessions 

Among other things I of course noticed that Postgres `huge_pages` was on the default `try` - which **IS NOT THE CORRECT VALUE for
large memory instances**. With `try`, if the OS has no huge pages reserved (which it by default doesn't), Postgres silently uses
normal 4 kB pages leading to bloated pagetables. With tens of GBs of `shared_buffers` the default page size means huge page tables, more TLB
misses and more memory access overhead for every single backend.

# What kind of improvement to expect?

After years of seeing the same I could have enabled and configured huge pages blindly...but as I did have some time budget left over
after achieving some typical quick wins:

* too aggressive checkpointing
* not aggressive enough Autovacuum
* no WAL compression
* bloated indexes
* unused indexes
* etc.

And given the company actually had invested in a basic load testing app for their business case (very rare to see something
like that sadly) - I did a quick benchmark on a spare machine...and the results: a solid **~12% TPS win out of thin air**
for their mixed (mostly OLTP) workload! Why not? I'd take it anytime. As that's actually a lot, considering that after common
quick wins (config tuning - background IO / write path optimizations, basic schema maintenance / bloat reductions) it becomes
increasingly hard to squeeze out anything without actually altering the schema or app code.

PS A pure `pgbench` read-only benchmark on an EC2 `m7gd.metal` instance showed even more - almost **20%** improvement, so YMMV of course and
best to check it out yourself. **I saw 900K vs 1.1M TPS** with a below "oneliner":  

```bash
pgbench -iq --unlogged -F 10 -s 1500 # Results in a 194 GB DB  
psql -c checkpoint
psql -c "create extension if not exists pg_prewarm" -c "select pg_prewarm('pgbench_accounts');" -c "select pg_prewarm('pgbench_accounts_pkey');"
psql -c "create extension if not exists pg_stat_statements" -c "select pg_stat_statements_reset();"
pgbench -S -M prepared -j 16 -c 192 -T 600 # Warmup / deplete bursting if any
time pgbench -S -M prepared -j 16 -c 192 -t 10000000 # Run 10 mio tx for all clients
psql -P x=on -P pager=off -c "select * from pg_stat_statements order by calls desc limit 1"
```

PS! Note that the difference in page table sizes (`grep ^PageTables /proc/meminfo`) was ~200x - reaching up to 20GB w/o 
huge pages configured, meaning basically 20GB less RAM for work / caching!

# Where does the bad advice come from?

Where does this "leaving it on default `try` is ok" advice seep in? It seems mostly from AI models...and overly lenient online config generators.
The generators IMHO could definitely be a lot stricter there. One could say `huge_pages=try` is the safe default, as it will work in any case - but that's just not a good
recommendation for a 256 GB+ memory box - and mostly when someone is using a config generator or LLM the chances are they're
setting up a new instance where it's OK to get an error at startup I think. 

*PS* The prompt I tested with a handful of common models was: "Perform Postgresql config tuning for a Linux instance with 64 vCPU, 256 GB RAM and 2TB of local SSD disks. Workload is OLTP".
There was some good advice in there though as well with hints at *huge_pages* + OS cooperation, but overall not great I think.

*PS2* I tested the following generators for the above listed hardware - here ordered by the quality of advice according to my world view:  

1. [PGTune](https://pgtune.leopard.in.ua/?dbVersion=18&osType=linux&dbType=oltp&cpuNum=64&totalMemory=256&totalMemoryUnit=GB&connectionNum=500&hdType=nvme&dbSize=mid_ram) - Good: shows as small "Note" box 
  stating basically "shared_buffers is large - you should do something"  
1. [pgconfigurator](https://pgconfigurator.cybertec-postgresql.com/tune?share=eyJ3IjoiT0xUUCIsInYiOjE4LCJyIjoyNzQ4Nzc5MDY5NDQsImMiOjY0LCJzIjoiTlZNZSIsIm1jIjoyNX0) by Cybertec - Satisfactory: leaves it at "try" but shows some extra (in a bit confusing wording perhaps) info when clicking on "huge_pages". Also seems to know that it's not a good idea to go over 32GB with  `shared_buffers` w/o guaranteed OS-side huge page support and they max out there.   
1. [mydba.dev](https://mydba.dev/tools/config-tuner) - Lacking: *Enable huge pages for large memory systems to reduce TLB misses. Requires OS-level configuration.* BUT no visual cues and "large systems" could mean different things to different people. 
1. [pgconfig.org](https://pgconfig.org/?max_connections=500&pg_version=18&environment_name=OLTP&total_ram=256&cpus=64&drive_type=SSD&arch=x86-64&os_type=linux) - Bad: "huge_pages" not even mentioned. 

# A warning on correct huge pages counting / sizing 

One, and pretty much the only thing to watch out when setting huge pages - **you should not overallocate** and do your calculations
[diligently](https://www.postgresql.org/docs/current/kernel-resources.html#LINUX-HUGE-PAGES)! As huge pages are carved out of normal memory and become a separate pool that the kernel uses only for hugetlb consumers.
Meaning - they cannot also be used for the regular page cache! To check against that - after PostgreSQL has started, *HugePages_Free*
from *meminfo* should be close to 0 (keeping a small margin is probably a good idea). 


# Bonus: min_dynamic_shared_memory

When configuring for a more parallel workload (lots of parallel queries spawning multiple workers) one could also consider
reserving some extra huge pages for `min_dynamic_shared_memory`, so that the dynamic shared memory segments used by parallel
queries can also be served from huge pages. I didn't test it for this OLTP use case though, and generally it can't be a huge
win there - but a good article explaining that (and plain `huge_page` nuances) is one from Bernd Helmle [here](https://www.cybertec-postgresql.com/en/huge-pages-postgresql/#dynamically-allocated-shared-memory-and-huge-pages).

*PS* Don't forget to disable Transparent Huge Pages and NUMA as well when already at it!

# Summary

Huge pages are not hard to configure and a few minutes of work will be rewarded with a nice profit - on large instances
the gain can be easily 10-20% for workloads with tons of concurrent sessions - avoid the defaults here and just configure
those huge pages and win!
