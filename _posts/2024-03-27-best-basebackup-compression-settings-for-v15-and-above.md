---
layout: post
title: Best pg_basebackup compression settings for v15 and above
cover-img: /assets/img/elephants_books_pg_basebackup_compression.jpg
tags: [postgres, pg_basebackup, backups, performance]
---

In my last [post](https://kmoppel.github.io/2024-01-05-best-pgdump-compression-settings-in-2024/) I did a quick check on
the performance of the newer (lz4, zstd) `pg_dump` compression options, which included setting up a small framework to
download some openly available "real life"-ish sample datasets. And the general result was that, indeed - the new algos
in lower levels provide the best value, especially *zstd*.

But `pg_dump` is about compressing essentially text based data...but how about binary Postgres data? Thus the tool to test
here additionally is `pg_basebackup`, with its newer (v15+) compression options. So let's see if something stands
out consistently again.

*TLDR;* - This time *lz4* in lowest compression level made the best impression on the tested 7 datasets.


# A prelude on pg_basebackup snapshots

Before going to test details - one might ask, are `pg_basebackup` snapshots even relevant nowadays? In short - yes, they are!

Especially the new server-side compression features for self-managed installations (you don't have access to the binary replication
stream when using managed). As when building replicas or doing PITR snapshots for huge databases, actually it is very often
the network that is the limiting factor. Even in 2024 the baseline (bursting won't help you much with a 5TB DB) network
bandwidth is commonly limited to 2 or 5 Gbps for lower and medium level SKUs, whereby local NVME disks can easily read 7 GB/s
and one could also stripe a few of these.

Also regular standalone "offsite" snapshots still make a lot of sense from a data safety standpoint - even if you have a backup system
like pgBackRest in place, it's always good to think about the "3-2-1" backup rule. And in such case why to congest the
network more than needed?

PS - also note that such "without WAL" `pg_basebackup` snapshots I took in this test, are only good together with a
previous WAL archiving or streaming set up.

```
pg_basebackup -c fast -Ft -D- -X none --no-manifest --compress=server-$METHOD:$LVL | wc -c
```

# The test setup

## Real-life datasets

As stitched together for the previous test, I used some publicly available real-life datasets and added
some Bash wrappers to download and restore them in a structured manner - in total the data amounted to 107 GB. The code
itself can be found on [Github](https://github.com/kmoppel/pg-open-datasets).

*PS* If you plan to run the script yourself, note that it will take around 2 days and download ~10GB over the internet

## Hardware

**Hardware**: 4 CPU Xeon E3 @ 3.30GHz, 16GB RAM, NVMe SSD

**OS**: Ubuntu 22.04 Server, `scaling_governor` in perf mode

**Postgres**: 16.2 from the official [PGDG](https://wiki.postgresql.org/wiki/Apt) repos, default config expect 4GB *shared_buffers*

**Measuring method**: A warmup run + 3 loops of `pg_basebackup | wc -c` for each compression method + level, meaning things
are cached, except for the largest "mouse_genome" dataset. With NVMe we're CPU-bound anyways though 

**Level of parallelism**: 1 core only

**Compression methods / levels**
```
METHOD_LVLS="gzip:1 gzip:3 gzip:5 gzip:7 lz4:1 lz4:3 lz4:5 lz4:7 zstd:1 zstd:5 zstd:9 zstd:13"
```
*PS* Note here that I left out some higher compression levels this time as one of the key learnings from the 1st test was
that they just lose to be efficient in any meaningful way - in this Postgres context at least of course.  

# Test results for gzip, lz4 and zstd  

## Evaluation criteria

Thing that was the hardest with this type of testing was to judge - what's a good backup? Just fast or rather well compressed? I think
there's no obvious "one-size-fits-all" winner here...and it will come down to requirements of specific databases.

Thus I went for the simplest - by equally prioritizing speed and output size. I normalized all test runs against both the shortest time
and smallest output byte size (in context of each dataset) and just summed the normalized values together to get a "score" or a "grade",
(where lower is better) for a particular method and level, per dataset. Meaning a perfect score would be 2 - having both the smallest
output and fastest execution and if something is very slow or produces a very large file, it won't float to the top.

I do hope this calculation flies, but just in case the [SQL](https://github.com/kmoppel/pg-open-datasets/blob/main/analyze_test_results_sql/compression_test_method_level_avg_rank.sql)
for you to check and ping me if very off.   

## By both speed + size normalized scores avg rank per dataset

Out of 12 different method + level combinations. 

|method|level|avg_per_dataset_rank|
|:----|:----|:----|
|lz4|1|1.3|
|zstd|1|1.7|
|zstd|5|3.1|
|lz4|3|4.0|
|gzip|1|5.0|
|lz4|5|6.3|
|gzip|3|6.7|
|gzip|5|8.7|
|lz4|7|9.0|
|zstd|9|9.1|
|gzip|7|11.3|
|zstd|13|11.7|



## Fastest per dataset 

|dataset_name|method|level|time_spent_s|avg_time_spent_s|backup_size|avg_backup_size|
|:----|:----|:----|:----|:----|:----|:----|
|imdb (8.5GB)|lz4|1|15.6|222.2|4088 MB|2812 MB|
|mouse_genome (65GB)|lz4|1|138.0|1136.9|21 GB|14 GB|
|nyc_taxi_rides (5.2GB)|lz4|1|12.5|140.0|2061 MB|1448 MB|
|osm_australia (6.2GB)|lz4|1|11.5|192.2|4633 MB|3748 MB|
|pgbench (15GB)|lz4|1|8.0|92.1|1640 MB|979 MB|
|postgrespro_demodb_big (2.6GB)|lz4|1|4.6|51.9|875 MB|624 MB|
|stackexchange_askubuntu (5.2GB)|lz4|1|11.1|132.6|2411 MB|1778 MB|


## Smallest output per dataset

|dataset_name|method|level|time_spent_s|avg_time_spent_s|backup_size|avg_backup_size|
|:----|:----|:----|:----|:----|:----|:----|
|imdb (8.5GB)|zstd|13|667.5|222.2|1986 MB|2812 MB|
|mouse_genome (65GB)|zstd|13|2970.1|1136.9|9021 MB|14 GB|
|nyc_taxi_rides (5.2GB)|zstd|13|522.0|140.0|1058 MB|1448 MB|
|osm_australia (6.2GB)|zstd|13|531.3|192.2|3447 MB|3748 MB|
|pgbench (15GB)|zstd|13|234.5|92.1|569 MB|979 MB|
|postgrespro_demodb_big (2.6GB)|zstd|13|167.9|51.9|468 MB|624 MB|
|stackexchange_askubuntu (5.2GB)|zstd|13|516.9|132.6|1356 MB|1778 MB|



## Summary

"Winners" for binary data compression are somewhat similar to the previous "textual" compression test, but there are also some differences.

* *lz4:1* was slightly ahead this time, as opposed to *zstd:1* for textual dumps. But in general *zstd* and *lz4* were
  more or less on par, and lowest compression levels for them clearly offer the best compromise between speed
  and output size for binary data - at least over the tested 7 datasets amounting to ~100 GB.
* *lz4:1* had constantly the fastest compression times, at the cost of approx 50% higher output sizes of course.
* *zstd:13* had constantly the highest compression rates, at ~3x time penalty.
* Higher compression level again really don't offer much at all, and maybe only to be considered for some ultra-long term archiving.
* My personal long-time default of *gzip* level 3 doesn't look that good at all anymore.
* On average the binary snapshots were much faster (~2-3x) than the dumps as one would expect...being at the same time
  though bigger by a similar factor on average.
* Note that this time I also did a few runs of manual restore on lowest compressions levels of all three algorithms...
  but the differences between them (<20%) seemed too little to get excited about - might only make a difference for some
  humongous DBs.
* Again I stumbled on an anomaly on highest tested level of *zstd* - the system actually went unresponsive for some reason
  and the `wc -c` pipe broke. Although it only happened once and worked on the re-run I'd nevertheless be wary of using higher
  levels of *zstd* for very critical instances.

  
*Bonus infomercial* - Need a hand with Postgres? I've been a Postgres-only engineer and consultant for 10+ yrs and some
topics I could help with are listed [here](https://kmoppel.github.io/aboutme/).
