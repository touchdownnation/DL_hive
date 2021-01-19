## Introduction

In this tutorial we’ll ingest datasets using Data Warehouse on Data Lake (CDH V.6)  We will also using sqoop service to offload data to hdfs thorght RDBMS such as Mysql databases in hive.

### DDL Operations

The Hive DDL operations are documented in Hive Data Definition Language.https://cwiki.apache.org/confluence/display/Hive/GettingStarted

Creating Hive Tables
---
creates a table called pokes with two columns, the first being an integer and the other a string.

    hive> CREATE TABLE pokes (foo INT, bar STRING);
    hive> CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING);
    
creates a table called invites with two columns and a partition column called ds. The partition column is a virtual column. It is not part of the data itself but  is derived from the partition that a particular dataset is loaded into.

 By default, tables are assumed to be of text input format and the delimiters are assumed to be ^A(ctrl-a). '\001'
 Browsing through Tables
    
    hive> SHOW TABLES;
lists all the tables.
---  
    hive> SHOW TABLES '.*s';    
 lists all the table that end with 's'. The pattern matching follows Java regular expressions. Check out this link for documentation  http://java.sun.com/javase/6/docs/api/java/util/regex/Pattern.html.

 shows the list of columns.

    hive> DESCRIBE invites;
 Altering and Dropping Tables
 ---
 Table names can be changed and columns can be added or replaced:

      hive> ALTER TABLE events RENAME TO 3koobecaf;
      hive> ALTER TABLE pokes ADD COLUMNS (new_col INT);
      hive> ALTER TABLE invites ADD COLUMNS (new_col2 INT COMMENT 'a comment');
      hive> ALTER TABLE invites REPLACE COLUMNS (foo INT, bar STRING, baz INT COMMENT 'baz replaces new_col2');  
 Note that REPLACE COLUMNS replaces all existing columns and only changes the table's schema, not the data. The table must use a native SerDe. REPLACE COLUMNS can also be used to drop columns from the table's schema:

      hive> ALTER TABLE invites REPLACE COLUMNS (foo INT COMMENT 'only keep the first column');
Dropping tables:
---
    hive> DROP TABLE pokes;
 
### DML Operations

The Hive DML operations are documented in Hive Data Manipulation Language. https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML  
Hive Data Manipulation Language.

Loading data from flat files into Hive:
---
    hive> LOAD DATA LOCAL INPATH './examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;   
Loads a file that contains two columns separated by ctrl-a into pokes table. 'LOCAL' signifies that the input file is on the local file system. If 'LOCAL' is omitted then it looks for the file in HDFS.

The keyword 'OVERWRITE' signifies that existing data in the table is deleted. If the 'OVERWRITE' keyword is omitted, data files are appended to existing data sets.
  
    hive> LOAD DATA LOCAL INPATH './examples/files/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
    hive> LOAD DATA LOCAL INPATH './examples/files/kv3.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-08');
The two LOAD statements above load data into two different partitions of the table invites. Table invites must be created as partitioned by the key ds for this to succeed.

    hive> LOAD DATA INPATH '/user/myname/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
The above command will load data from an HDFS file/directory to the table.
Note that loading data from HDFS will result in moving the file/directory. As a result, the operation is almost instantaneous.

### SQL Operations


SELECTS and FILTERS
---       
    hive> SELECT a.foo FROM invites a WHERE a.ds='2008-08-15';
selects column 'foo' from all rows of partition ds=2008-08-15 of the invites table. The results are not stored anywhere, but are displayed on the console.

Note that in all the examples that follow, INSERT (into a Hive table, local directory or HDFS directory) is optional.

    hive> INSERT OVERWRITE DIRECTORY '/tmp/hdfs_out' SELECT a.* FROM invites a WHERE a.ds='2008-08-15';    
selects all rows from partition ds=2008-08-15 of the invites table into an HDFS directory. The result data is in files (depending on the number of mappers) in that directory.
NOTE: partition columns if any are selected by the use of *. They can also be specified in the projection clauses.

Partitioned tables must always have a partition selected in the WHERE clause of the statement.
---
    hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/local_out' SELECT a.* FROM pokes a;
    
selects all rows from pokes table into a local directory.
---
    hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a;
    hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a WHERE a.key < 100;
    hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/reg_3' SELECT a.* FROM events a;
    hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_4' select a.invites, a.pokes FROM profiles a;
    hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT COUNT(*) FROM invites a WHERE a.ds='2008-08-15';
    hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT a.foo, a.bar FROM invites a;
    hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/sum' SELECT SUM(a.pc) FROM pc1 a;
    
selects the sum of a column. The avg, min, or max can also be used. Note that for versions of Hive which don't include HIVE-287, you'll need to use COUNT(1) in place of COUNT(*).

GROUP BY
 --- 
    hive> FROM invites a INSERT OVERWRITE TABLE events SELECT a.bar, count(*) WHERE a.foo > 0 GROUP BY a.bar;
    hive> INSERT OVERWRITE TABLE events SELECT a.bar, count(*) FROM invites a WHERE a.foo > 0 GROUP BY a.bar;
Note that for versions of Hive which don't include HIVE-287, you'll need to use COUNT(1) in place of COUNT(*).

JOIN
---    
    hive> FROM pokes t1 JOIN invites t2 ON (t1.bar = t2.bar) INSERT OVERWRITE TABLE events SELECT t1.bar, t1.foo, t2.foo;
MULTITABLE INSERT
  FROM src
  
    INSERT OVERWRITE TABLE dest1 SELECT src.* WHERE src.key < 100
    INSERT OVERWRITE TABLE dest2 SELECT src.key, src.value WHERE src.key >= 100 and src.key < 200
    INSERT OVERWRITE TABLE dest3 PARTITION(ds='2008-04-08', hr='12') SELECT src.key WHERE src.key >= 200 and src.key < 300
    INSERT OVERWRITE LOCAL DIRECTORY '/tmp/dest4.out' SELECT src.value WHERE src.key >= 300;

STREAMING
---  
    hive> FROM invites a INSERT OVERWRITE TABLE events SELECT TRANSFORM(a.foo, a.bar) AS (oof, rab) USING '/bin/cat' WHERE a.ds > '2008-08-09';
This streams the data in the map phase through the script /bin/cat (like Hadoop streaming).
Similarly – streaming can be used on the reduce side (please see the Hive Tutorial for examples).

For more information about the Compose file, see the https://cwiki.apache.org/confluence/display/Hive/GettingStarted

Data Type
---

### types primitive
- TINYINT
- SMALLINT
- INT
- BIGINT
- BOOLEAN ( TRUE/FALSE )
- FLOAT
- DOUBLE
- DECIMAL
- STRING
- VARCHAR
- TIMESTAMP ( YYYY-MM-DD HH:MM:SS.ffffffff )
- DATE ( YYYY-MM-DD )
```
cast ( string_column_value as FLOAT )
```
### types comples
* Arrays
```
array('a1', 'a2', 'a3')
```
* Structs
```
struct('a1', 'a2', 'a3')
```
* Maps
```
map('first', 1, 'second', 2, 'third', 3)
```
* Union
```
create_union
```


https://camo.githubusercontent.com/cb7b10848b318b6a7d5aebf6e625229462bac698807a0e5576a8ecd13aca1951/68747470733a2f2f7331392e706f7374696d672e63632f76673965717834777a2f62696773716c2d6461746174797065732e706e67


File Formatt
---
File Formats and Compression
File Formats
Hive supports several file formats:

- Text File
- SequenceFile
- RCFile
- Avro Files
- ORC Files
- Parquet
- Custom INPUTFORMAT and OUTPUTFORMAT 

![hiveffm](hiveFFM.png?raw=true "hivepicfm Logo")

Create table with define storage file formatt  
internal 
    
    CREATE TABLE IF NOT EXISTS testcreatetbl(
        `id` int, 	
        `user_` string, 	
        `group_` string, 	
        `file_count` bigint, 	
        `bytes` bigint, 	
        `raw_bytes` bigint, 	
        `hdfsname` string, 	
        `sample_date` bigint)	
    COMMENT 'thak test 2020'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS [--];
        
        -- TEXT, 
        -- SEQUENCEFILE
        -- PARQUETFILE
        -- ORC
        -- RCFILE
        -- TEXTFILE
     
external 

    CREATE EXTERNAL TABLE IF NOT EXISTS testcreatetbl(
        `id` int, 	
        `user_` string, 	
        `group_` string, 	
        `file_count` bigint, 	
        `bytes` bigint, 	
        `raw_bytes` bigint, 	
        `hdfsname` string, 	
        `sample_date` bigint)	
    COMMENT 'thak test 2020'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    STORED AS TEXTFILE
    location '/user/thak/data';

The hive.default.fileformat configuration parameter determines the format to use if it is not specified in a CREATE TABLE or ALTER TABLE statement.  Text file is the parameter's default value.

File Compression
---
- Compressed Data Storage
- LZO Compression


Managed vs. External Tables
---

Hive fundamentally knows two different types of tables:

- Managed (Internal)
- External

Feature comparison
     This means that there are lots of features which are only available for one of the two table types but not the other. This is an incomplete list of things:

- ARCHIVE/UNARCHIVE/TRUNCATE/MERGE/CONCATENATE  >> only work for managed tables
- DROP >> deletes data for managed tables while it only deletes metadata for external ones
- ACID/Transactional >> only works for managed tables
- Query Results Caching >> only works for managed tables


Managed tables
---
A managed table is stored under the hive.metastore.warehouse.dir path property, by default in a folder path similar to /user/hive/warehouse/databasename.db/tablename/. The default location can be overridden by the location property during table creation. If a managed table or partition is dropped, the data and metadata associated with that table or partition are deleted. If the PURGE option is not specified, the data is moved to a trash folder for a defined duration.

Use managed tables when Hive should manage the lifecycle of the table, or when generating temporary tables.

External tables
---
An external table describes the metadata / schema on external files. External table files can be accessed and managed by processes outside of Hive. External tables can access data stored in sources such as Azure Storage Volumes (ASV) or remote HDFS locations. If the structure or partitioning of an external table is changed, an MSCK REPAIR TABLE table_name statement can be used to refresh metadata information.

Use external tables when files are already present or in remote locations, and the files should remain even if the table is dropped.

