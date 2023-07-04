---
layout: post
title: TIL - IN is not the same as ANY
tags: [postgres, sql]
---

Not exactly from today, rather from a month or two ago, but still on my "noteworthy list". So after a remarkably long quiet period
of no surprises (Postgres doesn't generally surprise one badly), I managed to learn something controversial - a thing considered
generally good, using ANY instead of IN-list in this case, can have downsides nevertheless!

# Benefits of using ANY 

Reading the pertinent literature (for example [1](https://pganalyze.com/blog/5mins-postgres-performance-in-lists-vs-any-operator-bind-parameters),
[2](https://www.psycopg.org/psycopg3/docs/basic/adapt.html#lists-adaptation), [3](https://stackoverflow.com/questions/34627026/in-vs-any-operator-in-postgresql))
on the interwebs, talking to colleagues, or just from my personal experience - using ANY instead of IN-lists is a good thing!

It provides a few benefits like:

* You will get much less "noise" (only 1 entry) in the `pg_stat_statements` stats, to analyze possible performance problems more nicely
* Safer against potential SQL-injections with some frameworks
* A bit better performance compared to IN-lists for Postgres versions before v14 (fixed [here](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50e17ad281b8d1c1b410c9833955bc80fbad4078))
  * Note here though that previously I had never registered any meaningful performance differences before, and it's "just fine" for most cases to also use IN-lists   
* Avoids potential plan cache saturation
* More convenient with a bunch of text pattern matching filters - see for example the "Array of LIKEs" section
  [here](https://www.cybertec-postgresql.com/en/two-simple-postgres-tips-to-kick-start-year-2017/) that I once wrote about

# The surprising part of ANY

As in many cases - "borderline" scenarios / input values (think - none, zero, one, many) can be problematic - as in this case also.

So what did I learn?

That **IN and ANY can have huge execution time differences as they're basically not the same thing!** Although looking sneakily
very similar indeed. And they of course absolutely deliver the correct / same results in terms of output rows - but it can happen that
the chosen plans are very different, and there can be a penalty. As with a use case like described [here](https://www.postgresql.org/message-id/17922-1e2e0aeedd294424%40postgresql.org),
where we saw a massive, almost 5 orders of magnitude (~90000x !) difference on a production system for a seemingly harmless
syntax change of:

```sql
... WHERE project_id = ANY (ARRAY[1]::int8[])
```
vs
```sql
... WHERE project_id IN (1)
```

## Why's that?

For the interested - the explanation is that the IN syntax actually can get rewritten as `... WHERE project_id = 1` by the
parser, but for ANY there's no such logic in place due to some internal intricacies. 

 
# The solution

As seems plan behavioral parity is not on the horizon (as Tom Lane correctly noted, this is indeed not promised
anywhere in documentation also), I guess one should just be *aware* - and to be on the safe side for critical apps, one should
have some conditional logic in the app layer to **avoid using ANY for single input values**!

The confusing part though is that the documentation does [mention](https://www.postgresql.org/docs/current/functions-subquery.html#FUNCTIONS-SUBQUERY-ANY-SOME)
that *IN is equivalent to = ANY*, so that there are a few
articles on Stackoverflow, that misleadingly say that they are *exactly the same* (which is false as we saw), whereas I
"think" this refers only to the mathematical correctness of the query end result. 

# There's more to it actually

FYI - for completeness I should mention that there are a few other, more exotic I would say, differences / gotchas to
IN vs ANY behaviour, so you might want to check the below articles also for a complete understanding:
* [https://stackoverflow.com/questions/34627026/in-vs-any-operator-in-postgresql](https://stackoverflow.com/questions/34627026/in-vs-any-operator-in-postgresql)
* [https://dba.stackexchange.com/questions/125413/index-not-used-with-any-but-used-with-in](https://dba.stackexchange.com/questions/125413/index-not-used-with-any-but-used-with-in)
