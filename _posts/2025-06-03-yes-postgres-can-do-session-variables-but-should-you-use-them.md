---
layout: post
title: Yes, Postgres can do session vars - but should you use them?
cover-img: /assets/img/session-vars.jpeg
tags: [postgres, features, tables, howto]
---

Animated by some comments / complaints about Postgres' missing user variables story on a Reddit [post](https://www.reddit.com/r/PostgreSQL/comments/1kvkkro/postgresql_pain_points_in_real_world/)
about PostgreSQL pain points in the real world - I thought I'd elaborate a bit on sessions vars - which is indeed a little
known Postgres functionality.

Although this “alley” has existed for ages - and one can also use injected session variables to implement crazy stuff like token
based Row Level Security or store huge and complex JSON state, or just keep a bit of DB-side state over essentially stateless
statement-level connection pools - should you actually use it? What are the alternatives instead? Read on …

# What about temporary tables?

The obvious and more well known SQL way to keep some transient state is via temp tables! They give some nice data type
guarantees, performance, editor happiness to name a few benefits. But - **don’t use them for high frequency use cases!**
A few temp tables per second might already be too much and a disaster might be waiting to happen…Because `CREATE TEMP TABLE`
actually writes into system catalogs behind the scenes, which might not be directly obvious... And in cases of violent
mis-use - think frequent, short-lived temp tables with a lot of columns, plus unoptimized and overloaded Autovacuum together with long-running queries - can lead
to extreme catalog bloat (mostly on pg_attribute) and unnecessary IO for each session start / relcache filling / query planning.
And it’s also hard to recover from without some full locking - so that for critical high velocity DB’s it might be a good
idea to revoke temp table privileges altogether - for app / mortal users at least (not possible for superusers).

# Unlogged tables to the rescue

The 2nd most obvious way to keep some DB-side session state around would probably be to use more persistent normal tables, right?
Already better than temp tables as no danger of bloating the system catalog, right? NO. Pushing transient data though WAL
(including replicas and backup systems) is pretty bad and pointless and only to be recommended for tiny use cases.

In the Postgres world, **exactly for these kinds of transient use cases, special [UNLOGGED tables](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-UNLOGGED)
should be used!** Which can relieve the IO pressure on the system / whole cluster considerably.

One of course just needs to account for the semi-persistent nature - and the fact that they won’t be private anymore.
Meaning usage of RLS in case of secret data or just using some random enough keys to avoid collisions.

# Keeping short-term state with Postgres session variables

So finally to the actual point :)

As mentioned in the intro - Postgres actually supports private, non-persistent session vars! But is there a point to use
them, instead of app side vars / state keeping?

Rarely so actually …only when the app side code modifications are too hard or some process / managerial walls are hit...

But nevertheless a small usage primer:

## Using session level vars

Basically everything is similar to setting normal Postgres parameters like `statement_timeout`...but **one has to use a
custom scope prefix!** Which of course should differ from known extensions, which also crowd the same “space”.

```sql
-- SETTING
postgres=# SET myapp.x = '666' ;
SET

postgres=# SELECT set_config( 'myapp.x', '666', false) ;
 set_config 
------------
 666
(1 row)

-- READING
postgres=# SHOW myapp.x ;
 myapp.x 
---------
 666
(1 row)

postgres=# SELECT current_setting('myapp.x')::int ;
 current_setting 
-----------------
             666
(1 row)

```

## Making session vars handling more comfy

In the last example you probably saw that the downside of session vars is that it's not typed, but "text" only... and you need to ensure
correct handling yourself.

To combat that in a few cases I’ve actually gone as far as creating shorter, typed alias functions to extract the var - see
the example below. Not sure if I would recommend that today though, being explicit is generally a good thing.

```sql
CREATE FUNCTION uvi(key text) RETURNS int AS $$
  SELECT current_setting( key )::int
$$ LANGUAGE sql ;
```

Another gripe - for more complex cases / heavier usage it would be nice to be able to peek into `pg_settings` as well,
to see which custom variables have been set - but currently it’s not possible.

Also beware of another small inconsistency as well - although session vars are a textual thing, vars containing reserved keywords don't
work over SET / SHOW, so one needs to use the longer form functions to for example set something like:

```sql
SELECT set_config( 'x.table',  'part_202501', false);
```

# Benefits of session level vars again

* Allows for SQL queries / DB-side PL code to share or make something "globally available" for a session at low cost
* No IO performed
* No catalog churn guaranteed
* Works even when rights to create temp tables have been stripped
* Can store crazy long text strings “for free” (well, looking from some stingy low RAM app layer point of view) as Postgres “text” data type supports up to 1GB values
* Works also on replicas - haven’t seen the need for that in real life though

Hope it cleared the picture a bit.

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with setting up, migrating or troubleshooting Postgres for example. I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
