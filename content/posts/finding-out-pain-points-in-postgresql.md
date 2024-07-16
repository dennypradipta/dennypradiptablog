+++
title = 'Finding Out Pain Points in Postgresql using pg_stat_statements'
date = 2024-07-16T15:10:04+07:00
images = ['/finding-out-pain-points-in-postgresql.webp']
+++

Alright, let's face it: as much as we love PostgreSQL, it can sometimes feel like finding the source of your database's performance issues is like finding a needle in a haystack. But what if I told you there's an extension in your PostgreSQL toolbox that can help you identify and fix these pain points? Enter **pg_stat_statements**.

## What is pg_stat_statements?

It's an extension that collects and displays statistics about SQL queries executed by your database. It will show you which queries are causing trouble and need your attention.

## Getting Started

First things first, you need to enable the pg_stat_statements extension. Here’s how you do it:

```sql
CREATE EXTENSION pg_stat_statements;
```

Then, update your postgresql.conf. It is usually located in /etc/postgresql/(version)/main/postgresql.conf.Add the following line to your postgresql.conf file:

```text
shared_preload_libraries = 'pg_stat_statements'
log_statement = 'all'
log_min_duration_statement = 0
```

You’ll need to restart your PostgreSQL server for the changes to take effect.

## Finding the Culprit

Once you’ve got pg_stat_statements up and running, it’s time to uncover those pesky queries that are slowing things down. Here’s a handy query to get you started:

```sql
SELECT
    query,
    calls,
    total_exec_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent,
    mean_exec_time

FROM pg_stat_statements
ORDER BY mean_exec_time DESC;
```

(I got it from [here](https://www.timescale.com/learn/postgresql-extensions-pg-stat-statements), just modified it a bit)

This query retrieves various statistics about SQL statements from the pg_stat_statements view. Here’s what each part does:

1. `query`: This column shows the SQL statement that was executed. It's essential for identifying which queries are being analyzed.

2. `calls`: This column indicates how many times the query has been executed. A high number of calls can show which queries are run frequently and may be contributing significantly to the overall load on the database.

3. `total_exec_time`: This column gives the total time spent executing the query, in milliseconds. It’s useful for understanding the cumulative impact of the query on the database’s performance.

4. `rows`: This column shows the total number of rows retrieved or affected by the query. It helps in understanding the scale of the query's impact.

5. `hit_percent`: This calculated column shows the percentage of times a shared block was found in the buffer cache (a "cache hit") rather than having to be read from disk. It’s computed as:

6. `100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0)`: The nullif function is used to avoid division by zero. A higher percentage means better performance, as accessing data in memory is much faster than reading from disk.

7. `mean_exec_time`: This column shows the average time spent executing the query, in milliseconds.

Because I use [Metabase](https://www.metabase.com), I can just visually see the table. Metabase makes it easy to sort, filter, and visualize the data, turning what could be a complex analysis into a few clicks. So, instead of digging through raw data, I get a clear picture of what’s going on with my database.

![Metabase Dashboard](/finding-out-pain-points-in-postgresql-1.png)

## Taking Action

Identifying the slow queries is just the first step. Now, you need to roll up your sleeves and optimize them. Here are a few common tactics:

1. Indexing: Make sure your queries are using indexes effectively. Sometimes, a well-placed index can turn a slow crawl into a speedy sprint.
2. Query Optimization: Rewrite your queries for better performance. Sometimes just rephrasing can make all the difference.
3. Vacuuming: Regular maintenance tasks like vacuuming and analyzing your tables can help keep things running smoothly.

In my case, I forgot to index a few columns. _Oopsies_.

## Conclusion

Using pg_stat_statements is helpful to identify performance issues. It helps you pinpoint the exact queries that are causing the most pain, allowing you to take targeted action to improve your database's performance. So next time your PostgreSQL feels sluggish, give it a try.

And remember, if all else fails, there’s always coffee. Lots of coffee.
