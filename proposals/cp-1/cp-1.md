> Title: Support Incremental View Maintenance (IVM) in Cloudberry Database<br>
> Date: 07/24, 2023
> GitHub Discussions: https://github.com/orgs/cloudberrydb/discussions/36

### Proposers

- @avamingli
- @yjhjstz
- @my-ship-it

### Proposal Status

Under Discussion

### Abstract

Incremental View Maintenance (IVM) is a technique to maintain materialized views which computes and applies only the incremental changes to the materialized views rather than recomputing the contents as the current REFRESH command does. This feature is not implemented on PostgreSQL yet https://wiki.postgresql.org/wiki/Incremental_View_Maintenance.

IVM is more difficult and complex  in CBDB, IVM in a distributed database need several key technologies supported:
1. Change data capture(CDC)
 1.1 Based on Triggers on distributed system, catch data changed(insert/update/delete) has not been implemented.
 1.2 Based on Rules, update a table also update IVM if exists(Need to dig, didn't support COPY). 
 1.3 Based on DML logical replay (maybe enabled in utility mode on single segment, not been implemented in CBDB)
2. Complex expressions support
  Currently, built-in count sum, avg, min, and max are supported.
3. Distinct/duplicated rows in CBDB
4. Deferred View Maintenance
  When and how to auto refresh beside CDC.


### Motivation

Will benefit a lot, as description in [Snowflake Working with Materialized Views](https://docs.snowflake.com/en/user-guide/views-materialized)

>A materialized view is a pre-computed data set derived from a query specification (the SELECT in the view definition) and stored for later use. Because the data is pre-computed, querying a materialized view is faster than executing a query against the base table of the view. This performance difference can be significant when a query is run frequently or is sufficiently complex. As a result, materialized views can speed up expensive aggregation, projection, and selection operations, especially those that run frequently and that run on large data sets.

### Implementation

Draft branch https://github.com/cloudberrydb/cloudberrydb/tree/incremental_view_maintenance

### Rollout/Adoption Plan

_No response_