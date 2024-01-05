---
layout: post
title: Best pg_dump compression settings for Postgres in 2024
cover-img: /assets/img/til-when-was-my-postgres-cluster-initialized.png
tags: [postgres, pgdump, backups, performance]
---

Time between Christmas and New Year is always a good quiet time to be able to concentrate and tick off some mouldy TODO list items.
For me this time the top item was a script (which turned into a small framework somehow), to download and import some openly
available "real world" Postgres datasets to be able to perform some small tests on them.

The background for this "need" was that quite some time ago already when I was asked if the newer "generally good looking" compression
methods (lz4, zstd) added in recent Postgres versions are a no-brainer for a typical database or have also some hidden downsides and
should yet not serve as the new defaults. As I sadly didn't have a really good answer back then I had to go with the typical - it depends on the dataset....
But the topic piqued my interest and made me experiment a bit...but hell, how to get hands on that mythical "average" dataset
to benchmark against? Well, I guess the only way to get something of sorts would be to try to scrape together some real life
datasets and run some tests on them, and see if something stands out consistently!

*TLDR;* - *zstd* indeed should be the new default `pg_dump` compression method for most datasets.

FYI - note that `pg_dump` is not a great primary backup method against operational threats though!

## The test framework

The by-catch framework that I accidentally developed for this compression test can be found on [Github](https://github.com/kmoppel/pg-open-datasets)

*PS* If you plan to run the script yourself, note that it will take half a day and download ~7GB over the internet
and require ~100GB of disk space with default settings. If low on storage, set `DROP_DB_AFTER_TESTING=1` to drop all restored
DBs right after running the compression test.
	
# pg_dump compression test results for gzip, lz4 and zstd  

## Main evaluation criteria of "goodness"

This required a bit of head-scratching actually...how does best compression look like - fast or with smallest file sizes?   
As I'm not exactly a statistician I decided to ignore some more complex statistical models and go for the simplest thing that made
sense to - by equally prioritizing speed and output size. In short - I normalized all test runs against both the shortest time
and smallest output byte size (in context of each dataset) and just summed the normalized values together to get a "score" or a "grade",
where lower is better) for a particular method and level, per dataset. Meaning a perfect score would be 2 - having both the smallest
output and fastest execution.

I do hope this calculation flies, but just in case the [SQL](https://gist.github.com/kmoppel/3fe12db152fd38a0a98bd7de35bf7feb#file-pg_dump_compression_method_level_score-sql)
for you to check and ping me if very off.   

## By both speed + size normalized scores avg rank per dataset

| method | level | avg_per_dataset_rank |
|:-------|:------|:---------------------|
| zstd   | 1     | 1.0                  |
| zstd   | 5     | 2.2                  |
| gzip   | 1     | 4.5                  |
| lz4    | 1     | 5.3                  |
| gzip   | 3     | 5.7                  |
| lz4    | 3     | 6.2                  |
| zstd   | 9     | 7.7                  |
| lz4    | 5     | 8.0                  |
| gzip   | 5     | 8.0                  |
| lz4    | 7     | 9.7                  |
| gzip   | 7     | 11.0                 |
| zstd   | 13    | 11.7                 |
| lz4    | 9     | 12.2                 |
| gzip   | 9     | 12.8                 |
| zstd   | 17    | 15.0                 |
| lz4    | 11    | 15.0                 |
| zstd   | 21    | 17.0                 |


## Fastest per dataset 

| dataset_name            | method | level | time_spent_s | avg_time_spent_s | dump_size | avg_dump_size |
|:------------------------|:-------|:------|:-------------|:-----------------|:----------|-----------------|
| imdb                    | lz4    | 1     | 19.2         | 254.1            | 2004 MB   | 1315 MB         |
| mouse_genome            | lz4    | 1     | 209.6        | 1807.9           | 6644 MB   | 4211 MB         |
| osm_australia           | lz4    | 1     | 32.4         | 695.7            | 5434 MB   | 3655 MB         |
| pgbench                 | gzip   | 3     | 25.3         | 417.8            | 277 MB    | 278 MB          |
| postgrespro_demodb_big  | zstd   | 1     | 5.1          | 55.7             | 239 MB    | 256 MB          |
| stackexchange_askubuntu | lz4    | 1     | 10.5         | 259.5            | 1988 MB   | 1317 MB         |

## Smallest output per dataset

| dataset_name            | method | level | time_spent_s | avg_time_spent_s | dump_size | avg_dump_size |
|:------------------------|:-------|:------|:-------------|------------------|:----------|:--------------|
| imdb                    | zstd   | 13    | 225.2        | 254.1            | 1035 MB   | 1315 MB       |
| mouse_genome            | zstd   | 21    | 18540.3      | 1807.9           | 2261 MB   | 4211 MB       |
| osm_australia           | zstd   | 17    | 1829.0       | 695.7            | 2356 MB   | 3655 MB       |
| pgbench                 | gzip   | 9     | 59.8         | 417.8            | 263 MB    | 278 MB        |
| postgrespro_demodb_big  | zstd   | 21    | 409.1        | 55.7             | 169 MB    | 256 MB        |
| stackexchange_askubuntu | zstd   | 9     | 95.7         | 259.5            | 1008 MB   | 1317 MB       |

## Test hardware

Test machine this time was my workstation with an AMD Ryzen 5950X inside, Ubuntu Linux in perf CPU mode and with 32GB
of RAM - meaning all datasets except *mouse_genome* should be fully cached and the results not significantly disk affected.    

## Summary

After looking at the test (full SQL dump of my results table [here](https://gist.github.com/kmoppel/3fe12db152fd38a0a98bd7de35bf7feb#file-full_pg_dump_compression_test_results-sql)) I think one can conclude:

* Lower compression levels (1-5) for *zstd* offer the best bang for buck over all tested 6 datasets
* *zstd* also has the highest compression rates when using higher compression levels
* *lz4* in lower levels is the fastest option, at the cost of higher dump sizes of course (50-100% on average)
* Higher compression level really don't offer much besides warming the planet a bit
* My personal current Postgres `pg_dump` default compression of *gzip* level 3 is still an OK performer both time and size wise,
  fitting into the top bracket for all tested datasets
* The only artificial dataset in the bunch (pgbench) stood out well - it was the only one where *gzip* was the best
* Note that it was not a multi-core test and things could look differently for `pg_dump --format=directory` with a few *jobs*
  - The annoying thing (from benchmarking point of view) with the *directory* mode is though that we can't pipe to `/dev/null`
    and the filesystem writing will possibly start to play a role. Something like [nullfsvfs](https://github.com/abbbi/nullfsvfs)
    looks promising though to bypass that, need to test that
* Note that I didn't test the full range of compression levels but with a small stride, to minimize the runtime a bit
* Note that restore times were not tested (as it starts to depend a lot more on hardware and a few Postgres config settings)
  \- so if you indeed plan to use `pg_dump` for operational aspects, double-check
  that.
  - But also as noted in the intro, remember that dumps should not be the default backup strategy
* The test data showed that with *zstd* it can happen that higher compression levels actually increase the output size 🤯
  Which though to my surprising seems to be ["normal"](https://github.com/facebook/zstd/issues/3793)

# Request for Postgres open datasets suggestions 

Do you know of any cool real life "ready-to-load-into-postgres" datasets available on the interwebs? Please let me know or 
even better open a PR in the project [repo](https://github.com/kmoppel/pg-open-datasets)! Thank you!

FYI - as of new year I'm also up for some consulting on all matters around Postgres / databases in general. 