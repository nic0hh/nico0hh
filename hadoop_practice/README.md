# Hive/Hadoop practice

## Background

Hands-on exposure to Hive and the Hadoop ecosystem (HDFS, YARN, distributed querying), built through a self-directed practice environment rather than prior job experience with it.

Single-node Hadoop cluster on Google Cloud Dataproc, Hive on top of HDFS/YARN, everything run through the Hive CLI. Screenshots below are raw terminal output.

## Schema

Two tables, `employees` and `departments`, joined on department name.

![Create employees table](screenshots/01_create_employees_table.png)

![Insert employee rows](screenshots/02_insert_employees.png)

![Create departments table](screenshots/03_create_departments_table.png)

![Insert department rows](screenshots/04_insert_departments.png)

Even a small insert triggers a full Map/Reduce job under YARN. Hive queries are distributed jobs, not simple writes.

## Joins

Inner join, employees to department budget:

![Inner join result](screenshots/05_inner_join_result.png)

Left join, departments as the base table. Every department is kept, including one with no employees (NULL for its employee columns):

![Left join result](screenshots/06_left_join_result.png)

Table order matters here: whichever table follows `FROM` is preserved in full.

## Aggregation

Average salary per department, grouped with budget:

![Group by and average salary](screenshots/07_group_by_avg_salary.png)

Any selected column not wrapped in an aggregate function has to be in `GROUP BY`. Otherwise Hive has no single value to display once rows collapse into groups.

## Partitioning

Splits a table into separate physical folders by a chosen column, so a filtered query only scans the relevant folder instead of the whole table.

![Create partitioned table](screenshots/08_create_partitioned_table.png)

![Insert with dynamic partitioning](screenshots/09_insert_dynamic_partitions.png)

![Show partitions](screenshots/10_show_partitions.png)

Three partitions, one per department. The data was physically split, not just grouped in a query.

![Query against a single partition](screenshots/11_query_partition_pruning.png)

Filtering on `department = 'Engineering'` returns only the matching rows, with department reconstructed from the folder name rather than stored in the row itself.

## Limitations

- Small practice dataset, not production scale. Partitioning has no real performance payoff here. The point was understanding the mechanism, not benchmarking it.
- `EXPLAIN`/`EXPLAIN EXTENDED` didn't show a clean partition-pruning plan at this size. There's not enough data for the optimizer to do anything interesting with.
- Cluster setup itself involved real troubleshooting: zone capacity errors, a disk quota limit, Cloud Resource Manager API permissions. Resolved along the way.

## Takeaway

Joins (inner and left), aggregation, and partitioning: the core querying concepts that carry over from Hive to any SQL-on-Hadoop engine (Presto, Spark SQL).
