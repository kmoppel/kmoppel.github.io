---
layout: post
title: Striping Postgres data volumes - a free lunch?
cover-img: /assets/img/scaling.jpg
tags: [postgres, performance, ebs]
---

A small follow up on my [last post](https://kmoppel.github.io/2025-04-10-postgres-scaling-roadmap/) on various Postgres
scale-up avenues and my preferred order. The post received quite a bit of interest - meaning people at least still “wish”
performant databases :) And - the ones who are willing to put in a bit of effort with Postgres will also get it! Yeah,
that’s maybe the thing with Postgres - it’s a general purpose /  “Pareto” database engine basically - and not everything
is always handed out to you on a silver platter…

Anyways, after posting the article a few other tricks or approaches to scaling / performance ran through my head - especially
on the hardware level. For example a really nice performance optimization trick I think is **IO parallelization**! 
This requires OS access / self-management though, so not sure how applicable this is in the nowaday’s world…but this is
for example what RDS also does transparently in the background for larger volumes AFAIK. 

The concept is not something new of course - in the old spinning disk days every setup basically had at least a separate WAL
(called XLOG at that time) disk, commonly both (data + WAL) additionally RAID-ed for more speed. In many cases nowadays
you don’t need it anymore really, if to compensate with $$, i.e. managed IOPS or expensive volume types. But the same
approach actually could be still well used nowadays for cheap network storage where latencies generally are not at all on
par with locally attached disks.

As AWS is kind of representative of the public cloud I ran a few tests with various striping configurations.

*TLDR;* Striping together cheap volumes can indeed be considered a free lunch for some common usage patterns!


# Test setup


In short I threw together some [Bash + Ansible](https://github.com/kmoppel/ebs-striping-test) to create VMs with various
EBS striping configurations and run 3 read-only (writing ignored just to speed up testing and use UNLOGGED tables) workloads
using *pgbench*.

## Query patterns

* Pgbench default random key reads
* Custom indexed batch read of 10K sequential rows ~2MB
* Full scans

## Hardware / striping configuration

* VMs: 4 vCPU instances c6g.xlarge and m6idn.xlarge
* Stripes: None, 2, 4, 8, 16
* Stripe chunks size: 8, 16, 32, 64, 128 (kB)
* Dataset size: ~10x to RAM

To keep costs under control I used my Spot VM management utility called [PG Spot Operator](https://github.com/pg-spot-ops/pg-spot-operator) 
which makes cheap Postgres experimenting pretty effortless.

# Results

The first run results we’re a bit disappointing - and when I drilled into details, I saw that the selected cheapest 4 vCPU
instance of *c6g.xlarge* (selected by pg-spot-operator automatically based on the HW "wish-list") quickly reached the maximum
supported 20K IOPS limit on instance level, and things didn’t improve over 4 stripes. But after explicitly specifying a
better 100K max IOPS instance ([m6idn.xlarge](https://instances.vantage.sh/aws/ec2/m6idn.xlarge)) things seemed more logical
already.

Here a few graphs showing the effects of various striping settings:

*Random key reads TPS*
![random key reads TPS](/assets/img/ebs-striping-test/key_read_tps.png "random key reads TPS")

*Random key reads latency (ms)*
![random key reads latency](/assets/img/ebs-striping-test/key_read_latency_ms.png "random key reads latency in milliseconds")

*Batch reads TPS*
![batch reads TPS](/assets/img/ebs-striping-test/batch_read_tps.png "batch reads TPS")

*Batch reads latency (ms)*
![batch reads latency](/assets/img/ebs-striping-test/batch_read_latency_ms.png "batch reads latency in milliseconds")

*Full scan speed MB/s*
![full scan mbs](/assets/img/ebs-striping-test/full_scan_mbs.png "full scan MB/s")


Full test results SQL dump [here](https://gist.github.com/kmoppel/1af42008f3c2b649c18796db9091899b).


Hope it made you think, and feel free to [reach out](https://kmoppel.github.io/aboutme/) for my consulting services when need to scale, or at least want to be ready for it.
