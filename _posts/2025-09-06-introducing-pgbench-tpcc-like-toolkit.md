---
layout: post
title: A "TPC-C"-like "extension-pack" for pgbench
cover-img: /assets/img/pgbench-tpcc-like.jpg
tags: [postgres, benchmarking, pgbench, performance]
---

[TPC-C](https://en.wikipedia.org/wiki/TPC-C) is supposedly the most objective performance measurement of OLTP database systems...
and I've used it quite a fair bit over the years as well...yet, running it on Postgres is sadly not exactly as smooth as
things typically are with Postgres 😔 If to compare at least with the wonderful [pgbench](https://www.postgresql.org/docs/current/pgbench.html)
utility which ships as part of the project, and it's *tpcb-like* implementation - which sadly is considered largely obsolete
by now, and rightfully so...

# What is TPC-C about?

In short TPC-C simulates a retail company with a configurable number of warehouses, each with 10 districts and 3000
per-district customers and a pool of processed and new orders. Some inventory exists for the warehouses and customers
place orders composed of several items, which are limited to fixed 100K items. The transactions of course add new orders,
tracks payments, ship deliveries, check the inventory etc.

The approximate transaction mix as per the official [TPC-C spec](https://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp) is:

* New-Order: ~45%
* Payment: ~43%
* Order-Status: ~4%
* Delivery: ~4%
* Stock-Level: ~4%

And the ultimate performance metric reported is the number of orders processed per minute, or transactions-per-minute-C (*tpmC*).
Multiple parallel transactions are used to simulate the business activities, and each transaction is subject to response time
constraints and a few other intricate "keying" delays and data selection / value setting rules.

# Life in the "DBA trenches" 

In practice you're rarely interested in academical nuances and elaborate real-life modeling - but more often just want to
roughly validate some hardware, or test the effects of some *postgresql.conf* settings for tuning purposes - BUT still with a
real-life "looking" dataset and with more or less realistic data flows, i.e. a good mix of read and write and some limited
randomness. And you want it fast!

And this is where toolkits like [sysbench-tpcc](https://github.com/Percona-Lab/sysbench-tpcc), etc come in. Which already
simplifies things to a nice and practical level. There's a bunch of others of benchmarking tools out there as well of course,
but this is the one I've used mostly - *kudos* to the authors as well! Still...humans always
seem to find ways to complain and not be 100% happy with any given piece of software 😀 Welcome [pgbench-tpcc-like](https://github.com/kmoppel/pgbench-tpcc-like)!    

# Why yet another toolkit?

I've added some gripes with the "status quo" to the [README](https://github.com/kmoppel/pgbench-tpcc-like?tab=readme-ov-file#the-why)
as well, but the most poignant points could be:

* Most of the time I'm not really interested in academical correctness, but want to quickly make the database "sweat" heavily
  in a "close enough to real-life" way
* I'd prefer not to install any additional tools / libraries
* I want to be able to adjust the dataset size and generation parallelism at any time
* The data initialization phase should be fast, and the data generation should happen on the DB server, to avoid transmitting
  hundreds of gigabytes of data over the wire (as LibPQ doesn't do compression, [yet](https://commitfest.postgresql.org/patch/4746/)) 
* I want to be able to *easily* change read-write accent / weights, or throw in some custom queries / transactions
  to better account for the customer's actual workload.
 
# Quickstart

```bash
git clone https://github.com/kmoppel/pgbench-tpcc-like.git && cd pgbench-tpcc-like

createdb tpcc

psql -f 00_schema.sql tpcc 

# Generate ~40GB of starting data, using 4*100=400 "warehouses"
# 1 warehouse ~ 100MiB of disk space
pgbench -n -f 01_init_data.pgbench -c 4 -t 100 tpcc 

# Test for 30min with 32 clients / parallel sessions
pgbench -n -c 32 -T 1800 -P 300 -f new_order.pgbench@45 -f payment_transaction.pgbench@43 -f order_status.pgbench@4 \
  -f delivery_transaction.pgbench@4 -f stock_check.pgbench@4 tpcc

# Analyze results / stats, change configs / weights and rinse & repeat until happy :)
```
## Workload characteristics

*PS* Although not as write-heavy as the built-in `pgbench` workloads, (both the default *tpcb-like* and also *simple-update*)
the TPC-C workload is still pretty write heavy, with the following approximate distribution of queries - SELECT 66%,
UPDATE 18%, INSERT 15%, DELETE 1% - so you might want decide to increase the weights a bit for "order_status" and "stock_check"
transactions for more read-heavy / web use cases.

**All kinds of feedback very much appreciated!** 

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
