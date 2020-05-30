```sql
select txid_current();
INSERT into scott.employee VALUES (9,'avi',9);
select xmin,xmax,cmin,cmax,* from scott.employee where emp_id = 9;

BEGIN;
select txid_current();
DELETE from scott.employee where emp_id = 10;

CREATE EXTENSION pageinspect;
SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) 
FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |              tuple_data_split               
--------+--------+---------------------------------------------
    668 |      0 | {"\\x01000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x02000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x03000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x04000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x05000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x06000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x07000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x08000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x09000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x0a000000","\\x09617669","\\x01000000"}
(10 rows)

percona=# DROP TABLE scott.employee ;
DROP TABLE
percona=# CREATE TABLE scott.employee (emp_id INT, emp_name VARCHAR(100), dept_id INT);
CREATE TABLE
percona=# INSERT into scott.employee VALUES (generate_series(1,10),'avi',1);
INSERT 0 10
percona=# UPDATE scott.employee SET emp_name = 'avii';
UPDATE 10
percona=# SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) 
FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
    672 |    673 | {"\\x01000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x02000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x03000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x04000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x05000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x06000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x07000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x08000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x09000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x0a000000","\\x09617669","\\x01000000"}
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}

percona=# VACUUM scott.employee ;
VACUUM
percona=# SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) 
FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}


percona=# ANALYZE scott.employee ;
ANALYZE
percona=# select relpages, relpages*8192 as total_bytes, pg_relation_size('scott.employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
relpages | total_bytes | relsize 
---------+-------------+---------
6        | 49152       | 49152

```

As seen in the above examples, every such record that has been deleted but is still taking some space is called a dead tuple. Once there is no dependency on those dead tuples with the already running transactions, the dead tuples are no longer needed. Thus, PostgreSQL runs VACUUM on such Tables. VACUUM reclaims the storage occupied by these dead tuples. The space occupied by these dead tuples may be referred to as Bloat. VACUUM scans the pages for dead tuples and marks them to the freespace map (FSM). Each relation apart from hash indexes has an FSM stored in a separate file called <relation_oid>_fsm.

Whenever you delete a row, it’s not actually deleted, it is only marked as unavailable to all future transactions taking place after the delete occurs. The same happens with an update: the old version of a row is kept active until all currently running transactions have finished, then it is marked as unavailable. I emphasize the word unavailable because the row still exists on disk, it’s just not visible any longer. The VACUUM process in Postgres then comes along and marks any unavailable rows as space that is now available for future inserts or updates. The auto-vacuum process is configured to run VACUUM automatically after so many writes to a table (follow the link for the configuration options), so it’s not something you typically have to worry about doing manually very often (at least with more modern versions of Postgres).

People often assume that VACUUM is the process that should return the disk space to the file system. It does do this but only in very specific cases. That used space is contained in page files that make up the tables and indexes (called objects from now on) in the Postgres database system. Page files all have the same size and differently sized objects just have as many page files as they need. If VACUUM happens to mark every row in a page file as unavailable AND that page also happens to be the final page for the entire object, THEN the disk space is returned to the file system. If there is a single available row, or the page file is any other but the last one, the disk space is never returned by a normal VACUUM. This is bloat. Hopefully this explanation of what bloat actually is shows you how it can sometimes be advantageous for certain usage patterns of tables as well, and why I’ve included the option to ignore objects in the report.

Why bloat is actually a problem when it gets out of hand is not just the disk space it uses up. Every time a query is run against a table, the visibility flags on individual rows and index entries is checked to see if is actually available to that transaction. On large tables (or small tables with a lot of bloat) that time spent checking those flags builds up. This is especially noticeable with indexes where you expect an index scan to improve your query performance and it seems to be making no difference or is actually worse than a sequential scan of the whole table. And this is why index bloat is checked independently of table bloat since a table could have little to no bloat, but one or more of its indexes could be badly bloated. Index bloat (as long as it’s not a primary key) is easier to solve because you can either just reindex that one index, or you can concurrently create a new index on the same column and then drop the old one when it’s done.
