---
layout: post
title: Striping Postgres data volumes - a free lunch?
cover-img: /assets/img/ebs-striping-test/disks_wired.jpeg
tags: [postgres, performance, ebs]
---

A small follow up on my [previous post](https://kmoppel.github.io/2025-04-10-postgres-scaling-roadmap/) on various Postgres
scale-up avenues in my preferred order. The post received quite a bit of interest - meaning people at least still “wish”
performant databases :) And - the ones who are willing to put in a bit of effort with Postgres will also get it! Yes,
that’s the thing with Postgres - it’s a general purpose /  “Pareto” database engine basically - and not everything
is handed out to you on a silver platter…

Anyways, after posting the article a few other tricks or approaches to scaling / performance ran through my head - though
already more complex ones, like ultra-fast storage tablespaces for critical tables or indexes, and some similar
hardware / OS level tricks. For example one really nice performance optimization trick I think is **IO parallelization**! 
This requires OS access / self-management though, so not sure how applicable this is in the nowaday’s world…but this is
for example what RDS also does transparently in the background for larger volumes.

The concept is not something new of course - in the old days of spinning disks every setup basically had at least a separate WAL
(called XLOG at that time) disk, and commonly both (data + WAL, sometimes even logs!) additionally RAID-ed for more speed. In many cases nowadays
you don’t need it anymore really, if to compensate with $$, i.e. managed IOPS or expensive volume types. But the same
approach actually could still be applied nowadays to cheap network storage - where latencies generally are not at all on
par with locally attached disks...

As AWS is kind of representative of the public cloud I ran a few tests with various striping configurations, plus for
a comparison effect threw in my local disk setup and a cheap volume type in in-memory sauce also. See below for details.

**TLDR;** Striping together cheap volumes can indeed be considered a free lunch for some common usage patterns!


# Test setup


In short I threw together some [Bash + Ansible](https://github.com/kmoppel/ebs-striping-test) to create VMs with various
EBS striping configurations and run 3 read-only (writing ignored just to speed up testing and use UNLOGGED tables) workloads
using *pgbench*.

## Query patterns

* Pgbench default random key reads
* Custom indexed batch read of 10K rows (~2MB of sequential blocks)
* Full scans

## Hardware / striping configuration

* VMs: 4 vCPU instances c6g.xlarge and m6idn.xlarge
* Stripes: None, 2, 4, 8, 16
* Stripe chunks size: 8, 16, 32, 64, 128 (kB)
* Volume type: gp3 at default settings, i.e. 3K IOPS and 125MB/s throughput
* Dataset size: ~10x to RAM, i.e. pgbench scale 8000 for 16GB RAM m6idn.xlarge instance (plus fillfactor 80 to speed up a bit)
* Client count / parallelism: 16 for key read, 8 for batch, 1 for seq. scan

To keep costs under control I used my Spot VM management utility called [PG Spot Operator](https://github.com/pg-spot-ops/pg-spot-operator) 
which makes cheap Postgres experimenting pretty effortless.

# Results

The first run results we’re a bit disappointing - and when I drilled into details, I saw that the selected cheapest 4 vCPU
instance of *c6g.xlarge* (selected by pg-spot-operator automatically based on the requested minimum hardware specs) quickly reached the maximum
supported 20K IOPS limit on instance level, and things didn’t improve above 4 stripes. But after explicitly specifying a
better 100K max IOPS instance [m6idn.xlarge](https://instances.vantage.sh/aws/ec2/m6idn.xlarge) things seemed more logical
already.

Below a few graphs showing the effects of various striping settings. X-axis "4_16" for example means 4 striped volumes
with 16kB stripe size.

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

So in short one could say that a winning strategy would be "the more the merrier" here 🎉 And the stripe size sweet-spot
(for this particular test) seems to be 32kB, followed closely by 64kB default of LVM actually.

PS The graphs done via pgadmin4 didn't turn out too great, so a full test results SQL dump [here](https://gist.github.com/kmoppel/1af42008f3c2b649c18796db9091899b)
if someone wants better.

# Provisioned IOPS

Managing striping can be an annoyance, of course. The obvious alternative to increasing IO parallelisms is just to use
better, more expensive volume types plus [provisioned IOPS](https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops.html).

To get a comparative feel I also did:

* One run with *io2* + 40K provisioned IOPS.
* One run with *gp3* at default, BUT on a high 256GB RAM [x2iedn.2xlarge](https://instances.vantage.sh/aws/ec2/x2iedn.2xlarge)
  instance, so that effectively it's an in-memory test (which is rarely a real life use case of course)
* One run on my local workstation with a modern NVMe drive (2TB Samsung 990 EVO Plus). To get the same RAM-to-disk had to
  also 2x the pgbench scale to 16K.

The picture looked something like that:

| Volume config            | Key read TPS | Key read latency (ms) | Batch read TPS | Batch read latency (ms) | Full scan througput (MB/s) | 
|--------------------------|:------------:|----------------------:|:--------------:|:-----------------------:|:--------------------------:|
| gp3 16 stripes           |    28871     |                 1.053 |      510       |         27.737          |            2025            |
| io2 40k provisioned IOPS |    24338     |                 0.607 |      573       |         13.479          |            704             |
| local NVMe               |    48059     |                 0.267 |      866       |          9.26           |            1987            |
| single gp3 in-memory     |    147669    |                 0.012 |      1508      |          4.281          |            6789            |

So looking at comparable cloud volume setups only:

- *io2* with 40K guaranteed IOPS is somewhat better than 16 *gp3* stripes in latencies
- but basically on par with TPS
- and losing in total throughput
 
So in short I think I'd rather spend the [extra $2800](https://aws.amazon.com/ebs/pricing/) per month on something else :)

# Summary

When to ignore the extra management annoyance / risks of striping together a bunch of cheap *gp3* volumes - it is indeed
one of those rare "free lunch" strategies!

* Get much better latencies - went from 17ms (no striping) to ~1.1ms (16 stripes)
  * Mind here that the cheaper instances are capped at max 20K IOPS on instance level
  * Also remember it was a heavy random read + low caching use case, i.e. not your average case, so most probably the
    difference will be smaller for real life apps
* Get awesome throughput - saw a jump from 129MBs to 2GBs!
  * By the way, this is the main limiting factor with default / non-provisioned gp3 for most apps - not even HDD level scan speeds!
* Save quite a bit of money in the process!
  * With 16 stripes I got a no cost 48K IOPS - otherwise setting you back ~$3K a month!


PS Note that here one could go even crazier with stripe counts, as AWS allows a crazy high amount of those - on some
instances types even up to a whopping 128! At that point though the annual 0.1% failure rate could even come to haunt you
so better be prepared - for example, in case you didn't know -  **one can also do synchronized multi-volume snapshots
on EBS!**. Also, more expensive volume types provide a better annual failure rate as well, so probably can ignore the
topic then. And actually - never saw a failure on *gp3* anyways, and it’s still RDS Postgres default as well.

PS2 - not that the cloud is generally bad, but one needs to watch out up there, going with the "defaults", or perhaps
with the "default mindset" even, can lead to  overpaying quite a bit ...

*In need of a due-diligence check for your cloud databases? Feel free to [reach out](https://kmoppel.github.io/aboutme/) for my consulting
services!*
