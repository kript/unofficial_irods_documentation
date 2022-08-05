# Database tuning

The iCAT database is a key part of iRODS. In order to get iRODS to perform well, it is important to
ensure that you don't have any bottlenecks in your database configuration.

You can find generic advice about database tuning in [Use the Index, Luke](https://use-the-index-luke.com).

Some generic high-level advice:
* Newer database versions generally have performance improvements. If you have
  a very old database version, consider upgrading.
* If only some queries are very slow, verify that your iCAT database has
  indexes that the database can use to efficiently handle these queries. Indexes
  can potentially increase performance of particular queries a lot, although
  that can be at the expense of disk space and speed of adding new data.
* Ensure that your database has adequate resources, such as memory, and check
  that your database is configured to use these resources. Consider running
  your iCAT database on a separate server, so that it doesn't have to compete
  with other services for resources.

## Finding slow queries

One way to improve database performance is to first determine what queries
are taking a lot of time, and improve their performance (if they are used
frequently or getting them to run fast is important for other reasons).
PostgreSQL can be configured to log any slow queries using the
`log_min_duration_statement` setting. For example, in order to log any
queries that take more than one second (1000 ms), put the following in your
`postgresql.conf` and reload the configuration (`pg_ctl reload`):

```
log_min_duration_statement = 1000
```

An example of a logged statement could be:

```
LOG:  duration: 119619.149 ms  execute <unnamed>:
 select distinct R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,min(R_DATA_MAIN.create_ts ) ,max(R_DATA_MAIN.modify_ts ) ,R_DATA_MAIN.data_size from
 R_DATA_MAIN , R_COLL_MAIN  where upper (R_COLL_MAIN.coll_name ) LIKE $1  AND upper (R_DATA_MAIN.data_name ) LIKE $2  AND R_COLL_MAIN.coll_id =
 R_DATA_MAIN.coll_id  group by R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,R_DATA_MAIN.data_size  order by R_DATA_MAIN.data_name,
 R_COLL_MAIN.coll_name, R_DATA_MAIN.data_name
DETAIL:  parameters: $1 = '%COLL%', $2 = '%MYDATAOBJECT%'
```

This is a SELECT query that performs a case-insensitive wildcard search for data
objects with a particular collection and data object name.

## Finding the cause of slow queries

Generally speaking, it's advisable to test any queries and potential improvements
to make them run faster on a test server. When measuring query time, running
on a test server reduces the risk that query time is affected by other operations.
You can also test UPDATE, INSERT and DELETE queries without having to worry
about tests affecting your production database.

If you use PostgreSQL, you can use the [iCAT data generator](https://github.com/UtrechtUniversity/icat-data-generator)
to produce a database with synthetic data for performance testing.

If you find a query that takes a lot of time, you can retrieve the
query plan and statistics using the `EXPLAIN ANALYZE` command.

Example output on a test server with a smaller database:

```
icat_medium=# explain analyze select distinct R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,min(R_DATA_MAIN.create_ts ) ,max(R_DATA_MAIN.modify_ts ) ,R_DATA_MAIN.data_size from
 R_DATA_MAIN , R_COLL_MAIN  where upper (R_COLL_MAIN.coll_name ) LIKE '%BAR%'  AND upper (R_DATA_MAIN.data_name ) LIKE '%FOO%'  AND R_COLL_MAIN.coll_id =
 R_DATA_MAIN.coll_id  group by R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,R_DATA_MAIN.data_size  order by R_DATA_MAIN.data_name,
 R_COLL_MAIN.coll_name, R_DATA_MAIN.data_name;
                                                                               QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Unique  (cost=122590.55..122710.55 rows=8000 width=190) (actual time=1220.557..1236.456 rows=0 loops=1)
   ->  Sort  (cost=122590.55..122610.55 rows=8000 width=190) (actual time=1220.556..1236.455 rows=0 loops=1)
         Sort Key: r_data_main.data_name, r_coll_main.coll_name, (min((r_data_main.create_ts)::text)), (max((r_data_main.modify_ts)::text)), r_data_main.data_size
         Sort Method: quicksort  Memory: 25kB
         ->  Finalize GroupAggregate  (cost=121055.85..122071.92 rows=8000 width=190) (actual time=1220.541..1236.440 rows=0 loops=1)
               Group Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
               ->  Gather Merge  (cost=121055.85..121908.59 rows=6666 width=190) (actual time=1220.539..1236.437 rows=0 loops=1)
                     Workers Planned: 2
                     Workers Launched: 2
                     ->  Partial GroupAggregate  (cost=120055.82..120139.15 rows=3333 width=190) (actual time=1202.909..1202.912 rows=0 loops=3)
                           Group Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
                           ->  Sort  (cost=120055.82..120064.16 rows=3333 width=148) (actual time=1202.908..1202.910 rows=0 loops=3)
                                 Sort Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
                                 Sort Method: quicksort  Memory: 25kB
                                 Worker 0:  Sort Method: quicksort  Memory: 25kB
                                 Worker 1:  Sort Method: quicksort  Memory: 25kB
                                 ->  Nested Loop  (cost=0.43..119860.80 rows=3333 width=148) (actual time=1202.869..1202.871 rows=0 loops=3)
                                       ->  Parallel Seq Scan on r_coll_main  (cost=0.00..42088.00 rows=16667 width=105) (actual time=1202.868..1202.868 rows=0 loops=3)
                                             Filter: (upper((coll_name)::text) ~~ '%BAR%'::text)
                                             Rows Removed by Filter: 333333
                                       ->  Index Scan using idx_data_main3 on r_data_main  (cost=0.43..4.66 rows=1 width=59) (never executed)
                                             Index Cond: (coll_id = r_coll_main.coll_id)
                                             Filter: (upper((data_name)::text) ~~ '%FOO%'::text)
 Planning Time: 4.643 ms
 Execution Time: 1236.571 ms
(25 rows)
```

In this example, PostgreSQL has to scan all rows in the collection table (`r_coll_main`),
since it doesn't have a suitable index.

## Indexes

Since the query uses a wildcard search, we would need to add an index that supports
wildcard searches, like a [GIN index](https://www.postgresql.org/docs/11/gin.html).
If you don't have the `pg_trgm` extension installed yet, install it first:

```
icat_medium=# CREATE EXTENSION pg_trgm;
CREATE EXTENSION
```

Now add a matching index:
```
icat_medium=# CREATE INDEX idx_coll_gin1 ON r_coll_main USING gin (upper((coll_name)) gin_trgm_ops);
CREATE INDEX
```

This speeds up the query by a factor of roughly 100:

```
icat_medium=# explain analyze select distinct R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,min(R_DATA_MAIN.create_ts ) ,max(R_DATA_MAIN.modify_ts ) ,R_DATA_MAIN.data_size from
 R_DATA_MAIN , R_COLL_MAIN  where upper (R_COLL_MAIN.coll_name ) LIKE '%BAR%'  AND upper (R_DATA_MAIN.data_name ) LIKE '%FOO%'  AND R_COLL_MAIN.coll_id =
 R_DATA_MAIN.coll_id  group by R_DATA_MAIN.data_name ,R_COLL_MAIN.coll_name ,R_DATA_MAIN.data_size  order by R_DATA_MAIN.data_name,
 R_COLL_MAIN.coll_name, R_DATA_MAIN.data_name;
                                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Unique  (cost=106751.55..106871.55 rows=8000 width=190) (actual time=7.627..10.721 rows=0 loops=1)
   ->  Sort  (cost=106751.55..106771.55 rows=8000 width=190) (actual time=7.626..10.720 rows=0 loops=1)
         Sort Key: r_data_main.data_name, r_coll_main.coll_name, (min((r_data_main.create_ts)::text)), (max((r_data_main.modify_ts)::text)), r_data_main.data_size
         Sort Method: quicksort  Memory: 25kB
         ->  Finalize GroupAggregate  (cost=105216.85..106232.92 rows=8000 width=190) (actual time=7.621..10.714 rows=0 loops=1)
               Group Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
               ->  Gather Merge  (cost=105216.85..106069.59 rows=6666 width=190) (actual time=7.620..10.712 rows=0 loops=1)
                     Workers Planned: 2
                     Workers Launched: 2
                     ->  Partial GroupAggregate  (cost=104216.82..104300.15 rows=3333 width=190) (actual time=0.247..0.249 rows=0 loops=3)
                           Group Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
                           ->  Sort  (cost=104216.82..104225.16 rows=3333 width=148) (actual time=0.246..0.247 rows=0 loops=3)
                                 Sort Key: r_data_main.data_name, r_coll_main.coll_name, r_data_main.data_size
                                 Sort Method: quicksort  Memory: 25kB
                                 Worker 0:  Sort Method: quicksort  Memory: 25kB
                                 Worker 1:  Sort Method: quicksort  Memory: 25kB
                                 ->  Nested Loop  (cost=326.43..104021.80 rows=3333 width=148) (actual time=0.208..0.209 rows=0 loops=3)
                                       ->  Parallel Bitmap Heap Scan on r_coll_main  (cost=326.00..26249.00 rows=16667 width=105) (actual time=0.207..0.207 rows=0 loops=3)
                                             Recheck Cond: (upper((coll_name)::text) ~~ '%BAR%'::text)
                                             ->  Bitmap Index Scan on idx_coll_gin1  (cost=0.00..316.00 rows=40000 width=0) (actual time=0.019..0.020 rows=0 loops=1)
                                                   Index Cond: (upper((coll_name)::text) ~~ '%BAR%'::text)
                                       ->  Index Scan using idx_data_main3 on r_data_main  (cost=0.43..4.66 rows=1 width=59) (never executed)
                                             Index Cond: (coll_id = r_coll_main.coll_id)
                                             Filter: (upper((data_name)::text) ~~ '%FOO%'::text)
 Planning Time: 0.458 ms
 Execution Time: 10.800 ms

```

Beware that there is database overhead to maintain an index. For example, indexes take up disk space. So it's good
to check how much disk space is needed on a test server and consider this before adding indexes on production systems.

Finally, indexes can increase the time needed for inserting large amounts of data into the database. So if you want
to register a large amount of data in the database and be able to search it quickly afterwards, it can make sense
to insert the data first and add the indexes later. For example, if you are writing lots of checksums (e.g.
you have `msiDataObjChksum` in an upload rule), you might want to delay adding an index on the checksum
values until it actually becomes necessary to get adequate performance for checksum queries.

## Number of Connections

As iRODS does not do connection pooling, every connection to iRODS (including admin tasks) creates a new connection
to the database, so you need to be sure that your database is configured for a number of connections relative to
the amount of traffic you are expecting (and possibly a healthy overhead to be sure).
