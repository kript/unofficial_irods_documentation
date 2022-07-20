# Querying iRODS 

## Adding specific queries from the command line

bash escaping can cause havok when loading in a SQl query, so use this approach to work around that.
```
# create file e.g.
# vim lastWeekZeroLengthUnintentional.sql
# put that into an environment variable
QUERY=$(cat lastWeekZeroLengthUnintentional.sql )
# use the env varaible as a source for the query
iadmin asq "${QUERY}" lastWeekZeroLengthUnintentional
# run the query
iquest --no-page --sql lastWeekZeroLengthUnintentional
```

## Sending query output by email

```
iquest --no-page --sql lastWeekFilesSingleReplicaNotnoReplRoot | mail -s "Single Replica Files Created over the last week" -r "iRODS Bot irods@domain>" recipient@domain
```

## Specific queries to monitor catalog state

### Count of zero Length files where the checksum is *not* that of an empty file

```sql
select count(*) from r_data_main d where d.data_size = '0'
and to_timestamp(cast(d.create_ts as bigint)) > (NOW() - INTERVAL '7 DAY')
and d.data_id not in (select object_id from r_objt_metamap a, r_meta_main b
                        where a.meta_id = b.meta_id
                        and b.meta_attr_name = 'md5'
                        and b.meta_attr_value = 'd41d8cd98f00b204e9800998ecf8427e')
and d.resc_id not in (with recursive cte as (
                                                select r.resc_id, cast(r.resc_name as text) as resc_name, cast((case when r.resc_parent = '' then null else r.resc_parent end) as integer), 1 as level
                                                 from r_resc_main r
                                                union all
                                                select e.resc_id, c.resc_name || ';' || cast(e.resc_name as text) as resc_name,  cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer), c.level + 1
                                                from cte c
                                                join r_resc_main e on cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer) = c.resc_id
                                             )
                        select resc_id from cte z
                        where resc_name like 'noReplRoot%')
```
### Count of zero Length files created in the last week where the checksum is *not* that of an empty file

```sql
select count(*) from r_data_main d where data_size = '0'
and d.data_id not in (select object_id from r_objt_metamap a, r_meta_main b 
                        where a.meta_id = b.meta_id
                        and b.meta_attr_name = 'md5'  
                        and b.meta_attr_value = 'd41d8cd98f00b204e9800998ecf8427e')
and d.resc_id not in (with recursive cte as (
                                                select r.resc_id, cast(r.resc_name as text) as resc_name, cast((case when r.resc_parent = '' then null else r.resc_parent end) as integer), 1 as level
                                                 from r_resc_main r 
                                                union all
                                                select e.resc_id, c.resc_name || ';' || cast(e.resc_name as text) as resc_name,  cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer), c.level + 1
                                                from cte c
                                                join r_resc_main e on cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer) = c.resc_id
                                             )
                        select resc_id from cte z
                        where resc_name like 'noReplRoot%')
```

### List objects with a single replica where they are *not* in a tree that only has single replicas

A bit confusing this one, but impossible to do with GenQuery. The idea behind this query is that there are two trees in a Zone, one called 'root' and one called 'noReplRoot'.
As you might surmise, the 'root' tree contains a 'replicate' composite tree hierachy, so there should not be any files with a single replica unless they are in the noReplResc tree. 
This query lists objects violating those conditions created in the past week;

```sql
select a.data_id, b.coll_name, a.data_name, to_timestamp(cast(a.create_ts as bigint)) as create_date  from r_data_main a, r_coll_main b
where a.coll_id = b.coll_id
and to_timestamp(cast(a.create_ts as bigint)) > (NOW() - INTERVAL '7 DAY')
and a.data_id in 
(select d.data_id from r_data_main d where resc_id not in (
        with recursive cte as (
                select r.resc_id, cast(r.resc_name as text) as resc_name, cast((case when r.resc_parent = '' then null else r.resc_parent end) as integer), 1 as level
                from r_resc_main r 
                union all
                select e.resc_id, c.resc_name || ';' || cast(e.resc_name as text) as resc_name,  cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer), c.level + 1
                from cte c
                join r_resc_main e on cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer) = c.resc_id)
        select resc_id from cte z
        where resc_name like 'noReplRoot%')
group by d.data_id having count(*) = 1)
```

#### As above but with no time limit

```sql
select a.data_id, b.coll_name, a.data_name, to_timestamp(cast(a.create_ts as bigint)) as create_date  from r_data_main a, r_coll_main b
where a.coll_id = b.coll_id
and a.data_id in 
(select d.data_id from r_data_main d where resc_id not in (
        with recursive cte as (
                select r.resc_id, cast(r.resc_name as text) as resc_name, cast((case when r.resc_parent = '' then null else r.resc_parent end) as integer), 1 as level
                from r_resc_main r 
                union all
                select e.resc_id, c.resc_name || ';' || cast(e.resc_name as text) as resc_name,  cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer), c.level + 1
                from cte c
                join r_resc_main e on cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer) = c.resc_id)
        select resc_id from cte z
        where resc_name like 'noReplRoot%')
group by d.data_id having count(*) = 1)
order by a.create_ts desc
```

#### As above but no time limit and just return a count (for monitoring)

```sql
select count(*) from  (
        select d.data_id from r_data_main d where resc_id not in (
                with recursive cte as (
                select r.resc_id, cast(r.resc_name as text) as resc_name, cast((case when r.resc_parent = '' then null else r.resc_parent end) as integer), 1 as level
                 from r_resc_main r 
                union all
                select e.resc_id, c.resc_name || ';' || cast(e.resc_name as text) as resc_name,  cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer), c.level + 1
                from cte c
                join r_resc_main e on cast((case when e.resc_parent = '' then null else e.resc_parent end) as integer) = c.resc_id
                )
                select resc_id from cte z
                where resc_name like 'noReplRoot%')
group by d.data_id having count(*) = 1) t
```


