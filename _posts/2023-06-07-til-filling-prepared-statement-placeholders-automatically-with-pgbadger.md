---
layout: post
title: TIL - Filling prepared statement placeholders automatically with pgbadger
tags: [postgres, pgbadger, logfiles]
---

Hey Postgres friends! Thought I'll try out a new shorter broadcasting format as seems finding some hours for proper
blog posts is getting increasingly harder and harder. So here a first "Today I learned" type of post on a very useful
feature of [pgbadger](https://github.com/darold/pgbadger) that I just discovered for myself after all those years of
using pgbadger :bulb: 

# The problem

When some app uses the extended query protocol / prepared statements to query Postgres (which you generally should for performance reasons) and get
for some reason query errors or just cross the "slow query log" threshold (`log_min_duration_statement` config parameter),
you will not get full executable statements in the query log! But rather 2 pieces - a query with dollar-type placeholders
and after that a list of actual parameter values - that you then need to "fill in" if you want to actually re-run the
query (with EXPLAIN ANALYZE mostly) and see why this particular parameter values combination was slow for example.
 
# The solution

Previously if needing to re-run some parametrized prepared statements I did one of the following:

### Manual replace / fill

Inspect the logfile and fill in the blanks manually if it was a single or few queries with smallish amount of placeholders / 
parameters at play. No biggie, OK I guess to do some labor time to time.

### Use my ad-hoc script

When the amount of parameter was too large (10+) or dealing with many queries / executions, I looked where my home-brewn
param matching / filling Python script was located and hoped it would work for the particular logging configuration...
as it was a really simple regex matching line iterator, supporting only a few common `log_line_prefix` formats that I
mostly deal with.
  
## But now - pgbadger to the rescue!

Everyone getting hands-on with Postgres loves [pgbadger](https://github.com/darold/pgbadger), right? Me too.
But funnily - the only downside of this tool is that it has a **TON** of command-line parameters...I counted around a
hundred, not kidding!

So finally, on one lucky day recently I scrolled through all of those flags again for some other reason...and struck gold with:

```
--dump-all-queries     : dump all queries found in the log file replacing
                         bind parameters included in the queries at their
                         respective placeholders positions.
``` 

Voila!

### Demo usage

And how it looks in practice:

```
$ pgbench -i

$ PGOPTIONS='-c log_min_duration_statement=0' pgbench -M prepared -n -S -t 2

$ cat postgresql.log
...
2023-06-07 18:07:45.908 EEST [250162] LOG:  duration: 0.203 ms  parse P_0: SELECT abalance FROM pgbench_accounts WHERE aid = $1;
2023-06-07 18:07:45.908 EEST [250162] LOG:  duration: 0.220 ms  bind P_0: SELECT abalance FROM pgbench_accounts WHERE aid = $1;
2023-06-07 18:07:45.908 EEST [250162] DETAIL:  parameters: $1 = '93026'
2023-06-07 18:07:45.908 EEST [250162] LOG:  duration: 0.026 ms  execute P_0: SELECT abalance FROM pgbench_accounts WHERE aid = $1;
2023-06-07 18:07:45.908 EEST [250162] DETAIL:  parameters: $1 = '93026'
2023-06-07 18:07:45.908 EEST [250162] LOG:  duration: 0.020 ms  bind P_0: SELECT abalance FROM pgbench_accounts WHERE aid = $1;
2023-06-07 18:07:45.908 EEST [250162] DETAIL:  parameters: $1 = '26950'
2023-06-07 18:07:45.908 EEST [250162] LOG:  duration: 0.025 ms  execute P_0: SELECT abalance FROM pgbench_accounts WHERE aid = $1;
2023-06-07 18:07:45.908 EEST [250162] DETAIL:  parameters: $1 = '26950'

$ pgbadger --dump-all-queries postgresql.log
[========================>] Parsed 2165 bytes of 2165 (100.00%), queries: 4, events: 1

$ cat out.txt
...
SELECT abalance FROM pgbench_accounts WHERE aid = '93026';
SELECT abalance FROM pgbench_accounts WHERE aid = '26950';
```

**PS** Note here that if the pgbadger call fails or doesn't get the correct result, it might be that the log format autodetection
is in trouble (works correctly mostly though!) and needs some helping out via the `--prefix` parameter. 

**PS 2** Also note that actual prepared statement parameter values are not always guaranteed to be there, as they can be disabled for sensitive environments
via the relatively unknown `log_parameter_max_length` and `log_parameter_max_length_on_error` parameters - in that case of
course not even the almighty pgbadger with its 100 parameters can help you :smirk:
