# Performance Tuning for Developers

1. Performance Essentials

    * Oracle architecture basics
    * Performance metrics
    * Connecting to Oracle

1. Statement Level Tuning

    * Use of bind variables
    * Execution plan evaluation

1. Indexing

    * What makes a good index
    * What capabilities does Oracle provide

1. Application Monitoring

    * What is happening real time in oracle
    * What statements should target

sqlplus xe@localhost:1521 as sysdba

Create import directory:
CREATE OR REPLACE DIRECTORY import_dir AS 'C:\app\oracle-import';

show con_name for showing CDBS Common DBS.
show pdbs; to show PDBS where we can create new users etc

To change the db use this alter session set container = XEPDB1;

Create user/schema:
CREATE USER STUDENT3 identified by "Sella*2030";

Grant read and write to import_dir:
 GRANT READ, WRITE ON DIRECTORY import_dir TO student3;

## Why Performance Tuning Matters

1. Storage Array Network(SAN)

1. Oracle is memory intensive application, Oracle makes extensive use of caching.

1. Oracle has to have working areas of memory to perform operations like joins or sorts in order to full fill the sql statements.

### Oracle Architecure Overview

1. We can roughly divide memory used by oracle into two sections:

    * System Global Area(SGA) - 80% of the physical memory is used by SGA. Which is shared bu all users and all processes in oracle.
        * Shared Pool
        * Buffer Cache
    * Program Global Aread(PGA) - Used by the each session or login to the database. PGA is the private to a log in session.
        * Session Memory
            * SQL Work Area
            * Private SQL Area

1. Database Block - oracle is really working in blocks. A data block is the smallest level of granularity oracle will store data.

1. Block size can range from 2KB to 32 KB.

1. Database Block has Block Header and Rows.

1. Block Header which is going to contain some administrative data about the block. If the block is a table or index, what is the name of the object that the block contains data for?
    * Block type information
    * Table information
    * Row directory

1. oracle first try to locate the block then try to locate rows in the block.

1. Row Data
    * Some free space left at end of each row
    * Free space allows for insertion of new rows

### Buffer Cache

1. All SQL statements need to access blocks of data.

1. Buffer cache is used to cache blocks(both table and index) and most recently used data from disk.

1. Reading from memory much faster than disk.

1. Largest component of the SGA.

1. SQL statment will search the buffer cache for the blocks and read the required blocks, if the blocks are not available on the buffer cache, oracle will run physical IO operations on the disk to load blocks to buffer cache which can be used by the SQL statement.

1. As the preformance prespective we need to write queries which has very minimal physical IO operations, because physical IO is so much slower than reading from the Buffer Cache.

### Shared Pool

1. Shared pool is located in SGA. When you execute the SQL statement, Oracle first has to parse that statement, check to make sure the tabels and columns referenced in the statement exist.

1. Check permission on database objects and create a plan for how the oracle is going to execute the statement.

    * Data Dictionary Cache
        * Database object definitions
        * Object permissions
    * Shared SQL Area - which caches SQL statements and there execution plans
    * Reserved Pool

### Program Global Area

1. When ever an application connects to oracle, oracle creates a session for the connection. This session in oracle has memory associated with it, and this memory is part of what is called the Program Global Aread(PGA).

1. The memory asscociated with oracle session is private. This block of memory will only be used by its associated login session.

1. There are two primary areas in PGA, SQL Work Area and Private SQL Area.

1. SQL Work Area is a work area for any operations that need to be performed by SQL statements executed by this session. Operations like sort, hash join to geneate hash table were performed in SQL Work Area.

1. Private SQL Area has information about the Values of bind variables and query execution state information.

### Oracle Architecture Takeaways

1. Caching is extensively used

1. Write applications that take advantage of caching
    * Prefer memory operations over disk operations

### Performance Metrics

1. Elapsed Time - How long did a statement take to execute

1. Logical IO Operations - The number of logical IO is the number of read operations that a SQL statement performs from the Oracle buffer cache.

1. If the data block is not available in buffer cache, there will be a associated Physical IO operation for the statement.

1. Number of Logical IOs has impact on Physical IO, which effects the performance.
    * More likely need to read data from disk
    * Higher proability of reading more blocks.

1. Impacts CPU
    * More data to filter, join and sort
    * More data mean more CPU

1. Consistent Get = Logical IO

#### Measuring Logical IO Operations

1. Autotrace command
    * Available in SQL*Plus and SQL Developer
    * Develop performance baseline

1. V$ Views
    * Everytime oracle execute the statments, oracle keep the detailed statistics of the execution of that statement and these statics and summarized and availabe to you in a series of views.
    * Most notable view for us is V$SqlStats summarizes all statements
    * Identify which statements are most costly.
    * The data from the V$SqlStats, we can easily see what in the database are performing the most logical IOs.
    * You can also see other statstics, like the average amount of CPU time used, the average amount of elapsed time the statement took and the number of times a statement has been executed.

#### CPU Time

1. Oracle database servers are busy.

1. There are 2 main areas where a SQL statement uses CPU resources.

    * Statement Parsing
    * Statement Processing - Filtering, Joining, and Sorting

#### Performance Metrics Summary

1. Use metrics other than elapsed time to guide tuning efforts

1. Logical IOs help to understand statement efficiency

1. Think in terms of statement efficiency

### Performance Tuning Process

1. Analyze Statement

1. Tune SQL Statement

1. Measure Result of Change

## Connections and Connection Pools

1. In order to achive highest performance possible is by using connection pooling.

1. When an application uses connection pool, you keeping number of physical connections to the database open at all times.

1. When your application code needs a connection to the database, it will borrow one of these connections it has already openned from the connection pool.

### Connection Pool Mechanics

1. A connection pool is established for each connection string

1. Each connection pool has a minimum and maximum size

## Bind Variables

### Contention and Latch Waits

1. Latch - A serialization mechanism in Oracle used to synchronize access to shared resources.

1. Processes that are waiting to acquire the latch to access the shared SQL area are said to be experiencing latch waits.

1. Share SQL Area is accessed by only one process at the time of hard parse the SQL statement. Latch wait allow the other process to wait for the other process to finish the access.

1. Latch spinning

### Impacts of Shared SQL Area Contention

1. CPU is wasted as processes undergoing latch waits spin trying to acquire the latch.

1. Access to the shared SQL area will become a bottle neck
    * The more statements that are hard parsing, the worse the bottleneck will be
    * Think of a multi-lane highway merging down to a single lane
    * Response times will be slower because each process has to wait for access to the Shared SQL Area

1. A system will only run as fast as its slowest component.

### Takeaways From Bind Variables

1. Contention can occur even if only a few processes are hard parsing.

1. Applications that don't use bind variables consume an oversized amount of resources
    * Amount of CPU used and contention caused by one application will impact all applications

1. Failure to use bind variables limits the system throughput.

## Statement Level Performance Tuning

### Tuning SQL Statements for Performance

1. Look at the execution plan to understand what are the steps Oracle is performing to execute the SQL query.

1. Evaluates table statistics - How many records are available in the table and what indexes are available.

1. An execution plan of how to fulfill the SQL statement in the most efficient way.

### Execution Plan Defined

1. Execution Plan - The individual steps oracle will execute for a statement.

1. Each step is a basic operation oracle executes like, reading from a table or reading from a index, or performing a join.

1. Estimated Cost - The relative cost of this operation

### Getting Execution Plan

1. Commands to get execution plan

    ```sql
    EXPLAIN PLAN FOR SELECT * FROM TABLE;

    SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
    ```

### What does an execution plan contain ?

1. Operation Column - Operation performed by Oracle in each step, these are operations such as reading a table, reading a index, performaing a join or doing a sort.

1. Object Name Column - This column tells us the name of the database object, whether it is a table or index that oracle is accessing in this step of the execution plan.

1. cardinatlity - Estimate number of rows oracle expects to return from the step. This estimate depends on statistics oracle has for table or indexes involved in each step.

1. Cost - This is the estimate of how costly oracle thinks this operation will be to perform based on the amount of CPU and IO resources the operation will consume. Cost in the column inculdes the cost of any sub operations

1. Access_Predicates and Filter_predicates- have the join and column conditions in where clause.

### Reading an Execution Plan

1. We have to read the execution plan inside out. So we have to start at the innermost operation and work our way out from there.

### Autotrace Capability

1. We can use Autotrace capability to perform tuning on the SQL statements.

1. Autotrace will execute the statement, so be careful when you run auto trace on DML statements.

1. Autotrace will provide details like:
    * Number of Logical and physical IO's
    * Number of sorts
    * Network packets

### Logical IO Tuning Goals

1. Fewer logical IO operations means less data Oracle has to process.

1. Reducing logical IO will in trun reduce physical IO.

## Execution Plans in Depth

1. Table Operations
    * Table Scan Full
    * Table Access by Index Row ID

1. Index Operations
    * Index Range Scan
    * Index Unique Scan
    * Index Fast Full Scan
    * Index Full Scan

1. Join Operations
    * Nested Loops Join
    * Merge Join
    * Hash Join

### Table Access Operations

1. One of the primary drivers of performance.

1. We need to keep in mind, is that you want to sructure your SQL, where Oracle can retrive little data as possible. Try to retrive minimum amount of data.

#### Table Scan Full

1. Everybody in the database community refers to this as a Full Table Scan.

1. In this scenario, Oracle is going to read every block for the table and process each and every row, in every block.

1. Generally the most expensive operation.

1. Equivalent to doing a sequential scan through an array.

1. For large tables we should avoid Full Table Scan.

1. For small tables Full Table Scan is normal. Often more efficient than using an index.(efficient choice).

1. If the execution plan has Table Scan Full as the operation for small table, we no need to investigate - Oracle is functioning correctly.

#### Table Access by Index Row Id

1. In this case oracle has the Row IDs of the rows that are needed probably from an Index Lookup Operation that preceded this operation.

1. By having the ROW ID, oracle is able to first access the specified block, and then directly locate the required row inside the block that contains the needed data.

1. Oracle can view this because an Orcle Row ID, contains information both about the block where the data is stored, as well the exact location of the Row inside of the block of data.

1. This is very efficient, because oracle's able to directly go to the Row that contains the data that it's looking for.

#### Filter Predicates

1. Filter predicates are applied after the data is read.

1. Filter predicates on full table scans may indicate the need for an index.

1. Filter predicate on Table Access by Index Row id is applied on subset of data.

### Index Lookup Operations

#### Index Range Scan

1. The most common used index operation is Index Range Scan.

1. You will see Index Range Scan you when ever we use non-unique index to perform index lookup operation.

1. You will also see this operation used if you are accessing range of data on a unique index, either using between keyword or greather than, less than operators in the where clause.

#### Range Scan Through Leaf Nodes

1. Data is sorted order.

#### Index Unique Scan

1. If we have unqiue index and specify the value in the where clause, Oracle will perform Index Unique Scan.

1. Primary key in where clause use Index Unique Scan.

#### Index Access Predicates

1. Access Predicates are used with index operation in an execution plan.

1. When ever there is index range scan on index unique scan you will have access predicate associated with these operations.

1. Whenever oracle is using access predicate, it is using a condition in the where clause or a joined condition, in order to travesre the index and then only read the blocks it need to.

1. In situation where multiple rows could be returned like a non unique index or a where clause for a range of values the access predicate also determines the start and stop points of what blocks oracle has to read in the index.

#### Index Access and Filter Predicates

1. Access Predicate
    * Applied while reading the index
    * Limits amount of data read

1. Filter Predicate
    * Applied after index blocks have been read
    * Do not reduce the amount of data read

### Index Full Scan Operations

#### Index Fast Full Scan

1. Oracle is going to read the entire index, and it is going to read the index in unsorted order, as fast as possible.

1. Multi block reads are used like full table scan to read multiple blocks  to reduce the number of physical io operations that have to be performed for any blocks that are not already in the buffer pool.

1. Less efficient than index range scan

#### Index Full Scan

1. Oracle reads the entire contents of the index in sorted order, because ther is order by clause in the query and the every column used in order by caluse is contained in the index, and order of the columns in the order by clause should match the order of the columns in the index.

1. And second scenario is that query contains the group by clause, and all the columns in the groups by clause is contained in the index.

1. Single block reads are used as the blocks are read in order, that are not on the buffer pool.

1. Faster than table full scan.

### Join Operations

1. Nested Loop Operations, Hash Join, Merge Join

#### Nested Loop Operations

1. If both the tables are small nested loop operation is fine.

1. For each row in the driving source(Extranal Row Source) it will check each record of the inner source.

1. If the inner source is small, that will be acceptable for performance.

1. If there is no index on the inner row source, oracle has to lookup sequentially all the data in the inner row source.

#### Hash Join

1. When you are joining to large data sets hash join is used.

1. In Hash Join oracle choose one of the table sources that need to be joined, usually the smaller one and build a hash table from this row source.

1. Oracle will to perform a hash join when there is no suitable index that can be used in a nested loops join.

1. More physicall IO if the hashtable need to generated is big and not going to fit into temp space.

#### Neste Loops Join vs Hash Join

| Nested Loop Join | Hash Join |
| ---------------- | --------- |
|Up to 100 Million comparisons for 10,000 rows in each joining table|Cost to build the hash table|
||Cost to iterate through the second row source

#### Sort Merge Join

1. Sort merge join is used when the data in the two joining table are in sorted order and that sorted order matches the join condition that is applied.

#### Join Operations - Performance Questions

1. Which table will oracle will choose to access first, beacuse this table will drive any further join operations ?
Ans: Efficient for small tables, if the driving tables result set should be as small as possible to use nested loops join, other wise oracle will choose antoher join operation like Hash Join.

1. Are the joining columns indexed in both tables ?

1. Do I need perform this join?

### Tuning SQL Statements

1. Generate Execution Plan
1. Read Execution Plan
1. Understand Operations
1. Tune High Cost Operations

## Indexing Essentials

### Why Indexing Matters

1. How importance are indexes ?

    Ans: Most important factor influencing database performance. Poor indexing strategy equates to poor performance.

1. Why do indexes matter ?

    Ans:

    i. In oracle tabel, data is effectively randomly distributed.

    ii. Small fraction of data is required.

    iii. The resulting data access operation that we would have to perform is to read and process the entire table to find the data we need.

    What we need is some sort of data structure that help us navigate that data in the table, so we can more directly go to the data that we need, and this structure is an index.

1. What is an index ?

    Ans:

    i. Seperate data structure designed to improve retrieval of rows.

    ii. Each index is going to be defined by an `Index Key`, which is composed of subset of the columns in the table which are frequently used to retrive data from the table.

    iii. With this key we will store and oracle `Row ID` which is a pointer to the exact location in the table that actually contains data.

#### Index Costs

1. Duplication of data, Creation Costs, Maintenance Costs, Index must be kept in sync with data in table.

1. Index column order

1. Index selectivity

### B-Tree Indexes

1. Most common type of index.

1. B stands for balanced.

1. The data in the index is stored in sorted order. Why sorted order is so important, is that now we can build a tree structure on top of the sorted data.

1. We can build branch nodes from sorted data in leaf nodes.

1. Search O(log n), with B-Tree index in a table containing 1Million rows we can get required data in 19 comparisions.

### Bitmap Indexes

1. Useful for columns with a low number of distinct values(low cardinality) like (Gender, Marital Status)

1. The real power of bit map indexs comes when we can join several bitmap indexes together.

1. Targeted for data warehouse applications.

1. Do not use in OLTP databases, if we have to perform any DML operation on the column where Bitmap index is over the proces Oracle has to underfo to update the bitmap index is very expensive and very slow.

1. Only available in Oracle Enterprise Edition.

### Index Column Order Matters

1. Most of the time we will have index with multiple columns.

1. The order of the columns in the index matters for composite indexes.

1. If we have used tow non consecutive columns from the index in where clause, based on the leading edge column selectivty oracle will choose the column. A column is said to be highly selective if there is less number of rows matching that where clause. If there are less number of rows oracle probably will use the index and the second column value is used in filter predicate to the index keys, which is efficent when compared to full table scan.

### Index Skip Scan

1. Can still use an index operation

1. Can reduce overall number of indexes

### Index Selectivity

1. Measure of how discerning the index is

1. Selective indexes allow oracle to more directly find just the data it needs

1. Favor indexes with high selectivity

1. More selective indexes are indexes that are unique, primary key, email address, SSN.

1. Highly Selective index will have small number of matching records, so few table blocks read.

1. Poor selectivity index will have large number of matching records, most/all table block read.

1. Index Selectivity = Number of unique values / total number of rows in table.

1. Expected rows per index key = Total number of rows in table / Number of unique values(Flipping the index selectivity formula will give you the number of expeceted rows).

1. Index selectivity - using database statistics to check when stats where last gathered on table.

```sql
SELECT to_char(last_analyzed, 'YYYY-MM-DD hh24:mm')
FROM all_tables
WHERE owner = '<<schema_name>>' AND table_name = '<<table_name>>'
```

1. If you see the last_analyed date is too long, we need to gather statistics on table

```sql
-- Compute statistics on all rows
DBMS_STATS.GATHER_TABLE_STATS('<<schema_name>>', '<<table_name>>');

-- For 25 percent of the rows
DBMS_STATS.GATHER_TABLE_STATS('<<schema_name>>', '<<table_name>>', estimate_percent => 25);
```

### Selectivity in multi column index

1. Index selectivity = Number of unique values / Total number of rows in table

1. To find the number of unique values based on multi column:

```sql
select count(1) From (SELECT DISTINCT column1, column2 From table_name)
```

1. To get the unique values count of the columns in a table:

```sql
SLECT column_name, num_distinct
FROM all_tab_columns
WHERE owner = 'STUDENT'
AND table_name = 'APPLICATIONS'
ORDER BY num_distinct DESC
```

### Determing Index Column Order

1. Frequently used columns need to be at the front, will assure oracle can use the index

1. Pay attention to index selectivity

1. Favor frequency of use over selectivity

## Advance Indexing Techniques

### Covering Indexes

1. Typical Index Operation will find the matching ROWIDs in index, Look up rows in table via ROW ID.

1. All required data is found in the index, then that index is called coverining index and will skip the full table scan.

### Function Based Indexes

1. Index on a derived value.

```sql
SELECT * FROM STUDENTS WHERE UPPER(last_name) = UPPER('McNeil');
```

1. Creating a function based indext

```sql
CREATE INDEX ix_students_last_name
ON STUDENTS ( UPPER(last_name) );
```

1. Aggregate functions cannot be used COUNT(), MIN(), MAX(), AVG(), STDDEV() or any analytic functions.

### Indexing a subset of Rows using Function Based Index

1. Application Status where there are predefined values like (N, P, A, D).

1. Index only the rows intrested, to create index for subset of rows, use bellow query.

```sql
CREATE INDEX fx_applications_app_status
ON applications
(
    CASE application_status
        WHEN 'N' THEN 'N'
        WHEN 'P' THEN 'P'
        ELSE NULL
    END
)
```

1. The sql executed to select the above created index should be like bellow

```sql
SELECT * FROM applications
WHERE (
    CASE application_status
        WHEN 'N' THEN 'N'
        WHEN 'P' THEN 'P'
        ELSE NULL
    END
)
IN ('N', 'P')
```

1. If the case statement in the sql statement really bothers, we can create user defined function and use it in index and sql statement.

```sql
create or replace function map_application_status_value
(application_status_value IN VARCHAR)
return VARCHAR
DETERMINISTIC
IS mapped_value VARCHAR2(1);
BEGIN
    mapped_value :=
        CASE application_status
            WHEN 'N' THEN 'N'
            WHEN 'P' THEN 'P'
            ELSE NULL
        END;
    RETURN (mapped_value);
END;
```

1. Creating function based index using user defined function.

```sql
CREATE INDEX fx_application_app_status
ON applications
(map_application_status_value(application_status) );
```

1. Using same user defined function for where clause in the select statment to use the above created index.

```sql
SELECT * FROM applications
WHERE map_application_status_value(application_status)
IN ('N', 'P');
```

1. To know details of index use bellow query.

```sql
select ix.index_name, ix.distinct_keys, ix.leaf_blocks, ix.avg_leaf_blocks_per_key,
s.blocks, s.bytes
from all_indexes ix
INNER JOIN dba_segments s
ON ix.owner = s.owner
AND ix.index_name = s.segement_name
WHERE index_name = 'CREATED_INDEX_NAME';
```

### Index Compression

#### What is it ?

1. Index compression means, repeated values at the front of the index are compressed into a single prefix value.

#### How does this help ?

1. Storage space for the index is reduced, IO to read the index is reduced.

1. To create compressed index we need to use `COMPRESS` keyword in index creation.

```sql
CREATE INDEX ix_course_enroll_offering
ON course_enrollements
(course_offering_id, student_id)
COMPRESS 1;
```

1. For compressing multiple index columns we need to increase the number after `COMPRESS` keyword.

```sql
CREATE INDEX ix_course_enroll_offering
ON <<TABLE_NAME>>
(<<column_1>>, <<column_2>>, <<column_3>>)
COMPRESS 2;

-- In the above index we are compressing column_1 and column_2 together
```

### Invisible Indexes

1. Invisible indexes are the indexes which are not used by the oracle optimizer.

1. Can be used by specific sessions via a session parameter.

1. Index is maintained by oracle, still we have the overhead of maintaing the index, but not used in select queries.

#### What is the use of invisible index ?

1. Test New Index - Before making it public, test in dedicated session.

1. Dropping an Index - Soft drop the index, quickly trun back on if needed.

1. Creating an invisible index.

```sql
CREATE INDEX <<index_name>>
ON <<table_name>> (<<column_1>>, <<column_2>>)
INVISIBLE;
```

1. Change visibility of an index.

```sql
ALTER INDEX <<index_name>> INVISIBLE;

ALTER INDEX <<index_name>> VISIBLE;
```

1. Get current visibility of an index.

```sql
SELECT index_name, visibility
FORM all_indexes
WHERE index_name = '<<index_name>>';
```

1. Use invisible indexes for a session.

```sql
ALTER SESSION SET optimizer_use_invisible_indexes = true/false;
```

## Application Indexing Practices

### What should I Index ?

1. Frequntly used columns in where clause.

1. FK columns for join conditions.

### Indexing Costs and Overhead

1. Each index is a seperate data structure, which takes space on disk.

1. Takes memory when loaded into buffer cache.

1. To know the space used by index

```sql
SELECT ix.table_name, ix.index_name, s.blocks, s.bytes
FROM all_indexes ix
INNER JOIN dba_segments s
ON ix.index_name = s.segment_name
AND ix.table_owner = s.owner
WHERE ix.table_name = '<<table_name>>';
```

1. SAN(storage array network) is expensive.

1. Storage is cheap, fast storage is not.

1. Overhead in DML operations.

### Missing Leading Edge of Index

1. Check for the leading edge in the where clause, which is the first column in the index.

### Index Not Selective Enough

1. If the select query is not selective enough it will not use the index.

1. An index is said to be not selective enough if the query returns 10% of rows from the table data.

1. To solve this problem we need to imporve the selectivity of the index.

### Leading Wildcards In LIKE Clause

1. Oracle wont use the index if there is a leading wild cards in the like clause.

### Using Functions In WHERE Clause

1. We should have function based indexes on those cloumns which are used in function of a where clause.

### Implicit Data Type Cast

1. If we use the wrong data type for the column in where clause, oracle will not throw the error, but will do a implicit cast internally. In this case oracle will not use the index.

1. Make sure no implicit type conversions are occuring.

1. Use bind variables in the queries, explicitly specify data types with bind variable defenition. If we dont explicitly specigfy the data type oracle will do the best to find the correct data type based on the value used and the data type of the column.

### Database Statistics Up to Date

1. Stats are usually updated automatically by a night job configured by DBA in production system.

1. In other environments updating stats may not be updated automatically. We can update them using bellow two queries.

```sql
-- Update for one table and all indexes
EXEC DBMS_STATS.GATHER_TABLE_STATS (
    ownname => '<<Table Owner Name>>',
    tabname => '<<Table Name>>',
    estimate_percent => <<percent>>
);

-- Update for all objects in a schema
EXEC DBMS_STATS.GATHER_TABLE_STATS (
    ownname => '<<Table Owner Name>>'
);
```

## Monitoring Oracle Applications

### Queries for Monitoring

1. Who is logged in

```sql
SELECT username, osuser, program, module, machine, terminal, process,
to_char(logon_time, 'YYYY-MM-DD HH24:MMI:SS') AS logon_time, status,
CASE status WHEN 'ACTIVE' THEN NULL ELSE last_call_et END as idle_time
FROM v$session WHERE type = 'USER';
```

1. Where are my logins coming from

```sql
SELECT username, osuser, program, module, machine, process,
count(1) as login_count FROM v$session
WHERE type = 'USER'
GROUP BY username, osuser, program, module, machine, process;
```

1. What SQL is executing right now, which query in the session is blocking the other sessions data etc.

```sql
SELECT
    s.sid, s.username, s.osuser, s.machine, s.process, s.program, s.module,
    q.sql_text, q.optimizer_cost, s.blocking_session, bs.username as blocking_user,
    bs.machine as blocking_machine, bs.module as blocking_module,
    bq.sql_text as blocking_sql, s.event as wait_event, q.sql_fulltext
FROM v$session s
INNER JOIN v$sql q
    ON s.sql_id = q.sql_id
LEFT OUTER JOIN v$session bs -- blocking sessions
    ON s.blocking_session = bs.sid
LEFT OUTER JOIN v$sql bq -- blocking queries
    ON bs.sql_id = bq.sql_id
WHERE s.type = 'USER';
```

1. What statements consume the most resources(Finding the worst performing statements)

```sql
SELECT * FROM
(
    SELECT sql_id, sql_text, executions, elapsed_time,
    cpu_time, buffer_gets, disk_reads, 
    elapsed_time / executions as avg_elapsed_time,
    cpu_time / executions as avg_cpu_time,
    buffer_gets / executions as avg_buffer_gets,
    disk_reads / executions as avg_disk_reads
    FROM v$sqlstats
    WHERE executions  > 0
    ORDER BY elapsed_time / execution DESC
)
WHERE rownum <= 25;
```

## Useful Links

1. [Take your SQL from Good to Great - Common Table Expressions](https://towardsdatascience.com/take-your-sql-from-good-to-great-part-1-3ae61539e92a)
1. [Take your SQL from Good to Great - All about those dates](https://towardsdatascience.com/take-your-sql-from-good-to-great-part-2-cb03b1b7981b)
1. [Take your SQL from Good to Great - The other JOINs](https://towardsdatascience.com/take-your-sql-from-good-to-great-part-3-687d797d1ede)
1. [Take your SQL from Good to Great - Window Functions](https://towardsdatascience.com/take-your-sql-from-good-to-great-part-4-99a55fd0e7ff)