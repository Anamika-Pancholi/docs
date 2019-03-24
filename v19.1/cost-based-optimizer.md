---
title: Cost-Based Optimizer
summary: Learn about the cost-based optimizer
toc: true
redirect_from: sql-optimizer.html
---

The cost-based optimizer seeks the lowest cost for a query, usually related to time.

In versions prior to 2.1, a heuristic planner was used to generate query execution plans. The heuristic planner is only used in the following cases:

- If your query uses functionality that is not yet supported by the cost-based optimizer. For more information about the types of queries that are supported, see [Types of statements supported by the cost-based optimizer](#types-of-statements-supported-by-the-cost-based-optimizer).
- If you explicitly turn off the optimizer. For more information, see [How to turn the optimizer off](#how-to-turn-the-optimizer-off).

{% include {{ page.version.version }}/misc/beta-warning.md %}

## How is cost calculated?

A given SQL query can have thousands of equivalent query plans with vastly different execution times. The cost-based optimizer enumerates these plans and chooses the lowest cost plan.

Cost is roughly calculated by:

- Estimating how much time each node in the query plan will use to process all results
- Modeling how data flows through the query plan

The most important factor in determining the quality of a plan is cardinality (i.e., the number of rows); the fewer rows each SQL operator needs to process, the faster the query will run.

## View query plan

To see whether a query will be run with the cost-based optimizer, run the query with [`EXPLAIN (OPT)`](explain.html). The `OPT` option displays a query plan tree, along with some information that was used to plan the query. If the query is unsupported (i.e., it returns an error message that starts with e.g., `pq: unsupported statement` or `pq: aggregates with FILTER are not supported yet`), the query will not be run with the cost-based optimizer and will be run with the legacy heuristic planner.

For example, the following query (which uses [CockroachDB's TPC-H data set](https://github.com/cockroachdb/cockroach/tree/b1a57102d8e99b301b74c97527c1b8ffd4a4f3f1/pkg/workload/tpch)) returns the query plan tree, which means that it will be run with the cost-based optimizer:

{% include copy-clipboard.html %}
~~~ sql
> EXPLAIN (OPT) SELECT l_shipmode, avg(l_extendedprice) from lineitem GROUP BY l_shipmode;
~~~

~~~
                                     text
+-----------------------------------------------------------------------------+
group-by
├── columns: l_shipmode:15(string!null) avg:17(float)
├── grouping columns: l_shipmode:15(string!null)
├── stats: [rows=700, distinct(15)=700]
├── cost: 1207
├── key: (15)
├── fd: (15)-->(17)
├── prune: (17)
├── scan lineitem
│    ├── columns: l_extendedprice:6(float!null) l_shipmode:15(string!null)
│    ├── stats: [rows=1000, distinct(15)=700]
│    ├── cost: 1180
│    └── prune: (6,15)
└── aggregations [outer=(6)]
└── avg [type=float, outer=(6)]
└── variable: l_extendedprice [type=float, outer=(6)]
(16 rows)
~~~

In contrast, queries that are not supported by the cost-based optimizer return errors that begin with the string `pq: unsupported statement: ...` or specific messages like `pq: aggregates with FILTER are not supported yet`. Such queries will use the legacy heuristic planner instead of the cost-based optimizer.

## Types of statements supported by the cost-based optimizer

The cost-based optimizer supports most SQL statements. Specifically, the following types of statements are supported:

- [`CREATE TABLE`](create-table.html)
- [`UPDATE`](update.html)
- [`INSERT`](insert.html), including:
  - `INSERT .. ON CONFLICT DO NOTHING`
  - `INSERT .. ON CONFLICT .. DO UPDATE`
- [`UPSERT`](upsert.html)
- [`DELETE`](delete.html)
- `FILTER` clauses on [aggregate functions](functions-and-operators.html#aggregate-functions)
- [Sequences](create-sequence.html)
- [Views](views.html)
- All [`SELECT`](select.html) statements that do not include window functions
- All `UNION` statements that do not include window functions
- All `VALUES` statements that do not include window functions

This is not meant to be an exhaustive list. To check whether a particular query will be run with the cost-based optimizer, follow the instructions in the [View query plan](#view-query-plan) section.

## Table statistics

The cost-based optimizer can often find more performant query execution plans if it has access to statistical data on the contents of your database's tables. This statistical data needs to be generated from scratch for new tables, and regenerated periodically for existing tables.

{% include {{ page.version.version }}/misc/automatic-statistics.md %}

To manually generate statistics for a table, run a [`CREATE STATISTICS`](create-statistics.html) statement like the one shown below. It automatically figures out which columns to get statistics on &mdash; specifically, it chooses:

- Columns that are part of the primary key or an index (in other words, all indexed columns).
- Up to 100 non-indexed columns.

Note that the above also describes the statistics gathered by the automatic statistics feature, since it runs a query similar to the one shown below.

{% include copy-clipboard.html %}
~~~ sql
> CREATE STATISTICS employees_stats FROM employees;
~~~

{{site.data.alerts.callout_info}}
Every time the [`CREATE STATISTICS`](create-statistics.html) statement is executed, it kicks off a background job. For more information, see [View statistics jobs](create-statistics.html#view-statistics-jobs).
{{site.data.alerts.end}}

## Query plan cache

<span class="version-tag">New in v19.1</span>: CockroachDB uses a cache for the query plans generated by the optimizer. This can lead to faster query execution since the database can reuse a query plan that was previously calculated, rather than computing a new plan each time a query is executed.

The query plan cache is enabled by default. To disable it, execute the following statement:

{% include copy-clipboard.html %}
~~~ sql
> SET CLUSTER SETTING sql.query_cache.enabled = false;
~~~

Finally, note that only the following statements use the plan cache:

- [`SELECT`](select.html)
- [`INSERT`](insert.html)
- [`UPDATE`](update.html)
- [`UPSERT`](upsert.html)
- [`DELETE`](delete.html)

## Join reordering

<span class="version-tag">New in v19.1</span>: The cost-based optimizer will explore additional join orderings in an attempt to find the lowest-cost execution plan for a query involving multiple joins, which can lead to significantly better performance in some cases.

Because this process leads to an exponential increase in the number of possible execution plans for such queries, it's only used to reorder subtrees containing 4 or fewer joins by default.

To change this setting, which is controlled by the `experimental_reorder_joins_limit` [session variable](set-vars.html), run the statement shown below. To disable this feature, set the variable to `0`.

{% include copy-clipboard.html %}
~~~ sql
> SET experimental_reorder_joins_limit = 6;
~~~

{{site.data.alerts.callout_danger}}
We strongly recommend not setting this value higher than 8 to avoid performance degradation. If set too high, the cost of generating and costing execution plans can end up dominating the total execution time of the query.
{{site.data.alerts.end}}

For more information about the difficulty of selecting an optimal join ordering, see our blog post [An Introduction to Join Ordering](https://www.cockroachlabs.com/blog/join-ordering-pt1/).

## Join hints

<span class="version-tag">New in v19.1</span>: The optimizer supports hint syntax to force the use of a specific join algorithm. The algorithm is specified between the join type (`INNER`, `LEFT`, etc.) and the `JOIN` keyword, for example:

- `INNER HASH JOIN`
- `OUTER MERGE JOIN`
- `LEFT LOOKUP JOIN`
- `CROSS MERGE JOIN`

Note that the hint cannot be specified with a bare hint keyword (e.g., `MERGE`) - in that case, the `INNER` keyword must be added. For example, `a INNER MERGE JOIN b` will work, but `a MERGE JOIN b` will not work.

{{site.data.alerts.callout_info}}
Join hints cannot be specified with a bare hint keyword (e.g., `MERGE`) due to SQL's implicit `AS` syntax. If you're not careful, you can make `MERGE` be an alias for a table; for example, `a MERGE JOIN b` will be interpreted as having an implicit `AS` and be executed as `a AS MERGE JOIN b`, which is just a long way of saying `a JOIN b`. Because the resulting query might execute without returning any hint-related error (because it is valid SQL), it will seem like the join hint "worked", but actually it didn't affect which join algorithm was used. In this case, the correct syntax is `a INNER MERGE JOIN b`.
{{site.data.alerts.end}}

### Supported join algorithms

- `HASH`: Forces a hash join; in other words, it disables merge and lookup joins. A hash join is always possible, even if there are no equality columns - CockroachDB considers the nested loop join with no index a degenerate case of the hash join (i.e., a hash table with one bucket).

- `MERGE`: Forces a merge join, even if it requires re-sorting both sides of the join.

- `LOOKUP`: Forces a lookup join into the right side; the right side must be a table with a suitable index. Note that `LOOKUP` can only be used with `INNER` and `LEFT` joins.

If it is not possible to use the algorithm specified in the hint, an error is signaled.

### Additional considerations

- This syntax is consistent with the [SQL Server syntax for join hints](https://docs.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-join?view=sql-server-2017), except that:

   - SQL Server uses `LOOP` instead of `LOOKUP`.

   - CockroachDB does not support `LOOP` and instead supports `LOOKUP` for the specific case of nested loop joins with an index.

- When a join hint is specified, the two tables will not be reordered by the optimizer. The reordering behavior has the following characteristics, which can be affected by hints:

   - Given `a JOIN b`, CockroachDB will not try to commute to `b JOIN a`. This means that you will need to pay attention to this ordering, which is especially important for lookup joins. Without a hint, `a JOIN b` might be executed as `b INNER LOOKUP JOIN a` using an index into `a`, whereas `a INNER LOOKUP JOIN b` requires an index into `b`.

   - `(a JOIN b) JOIN c` might be changed to `a JOIN (b JOIN c)`, but this does not happen if `a JOIN b` uses a hint; the hint forces that particular join to happen as written in the query.

## How to turn the optimizer off

With the optimizer turned on, the performance of some workloads may change. If your workload performs worse than expected (e.g., lower throughput or higher latency), you can turn off the cost-based optimizer and use the heuristic planner.

To turn the cost-based optimizer off for the current session:

{% include copy-clipboard.html %}
~~~ sql
> SET optimizer = 'off';
~~~

To turn the cost-based optimizer off for all sessions:

{% include copy-clipboard.html %}
~~~ sql
> SET CLUSTER SETTING sql.defaults.optimizer = 'off';
~~~

{{site.data.alerts.callout_info}}
Changing the cluster setting does not immediately turn the optimizer off; instead, it changes the default session setting to `off`. To see the change, restart your session.
{{site.data.alerts.end}}

## Known limitations

- Some features are not supported by the cost-based optimizer; however, the optimizer will fall back to the heuristic planner for this functionality. If performance is worse than in previous versions of CockroachDB, you can [turn the optimizer off](#how-to-turn-the-optimizer-off) to manually force it to fallback to the heuristic planner.
- Some [correlated subqueries](subqueries.html#correlated-subqueries) are not supported by the cost-based optimizer yet. If you come across an unsupported correlated subquery, please [file a Github issue](file-an-issue.html).

## See also

- [`SET (session variable)`](set-vars.html)
- [`SET CLUSTER SETTING`](set-cluster-setting.html)
- [`SHOW (session variable)`](show-vars.html)
- [`CREATE STATISTICS`](create-statistics.html)
- [`SHOW STATISTICS`](show-statistics.html)
- [`EXPLAIN`](explain.html)