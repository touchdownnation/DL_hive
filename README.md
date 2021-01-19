Introduction
==============
![hive](buzzbee2.png?raw=true "hivepic Logo")

In this tutorial we’ll ingest datasets using Data Warehouse on Data Lake (CDH V.6)  We will also using sqoop service to offload data to hdfs thorght RDBMS such as Mysql databases in hive.




DDL Operations

Creating Hive Tables
creates a table called pokes with two columns, the first being an integer and the other a string.

    hive> CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING);
     




For more information about the Compose file, see the https://cwiki.apache.org/confluence/display/Hive/GettingStarted

    hive> CREATE TABLE pokes (foo INT, bar STRING);
    hive> CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING);
    
 creates a table called invites with two columns and a partition column called ds. The partition column is a virtual column. It is not part of the data itself but  is derived from the partition that a particular dataset is loaded into.

 By default, tables are assumed to be of text input format and the delimiters are assumed to be ^A(ctrl-a).
 Browsing through Tables
    
    hive> SHOW TABLES;

lists all the tables.
  
    hive> SHOW TABLES '.*s';
    
 lists all the table that end with 's'. The pattern matching follows Java regular expressions. Check out this link for documentation  http://java.sun.com/javase/6/docs/api/java/util/regex/Pattern.html.

 shows the list of columns.

    hive> DESCRIBE invites;

 Altering and Dropping Tables
 Table names can be changed and columns can be added or replaced:

      hive> ALTER TABLE events RENAME TO 3koobecaf;
      hive> ALTER TABLE pokes ADD COLUMNS (new_col INT);
      hive> ALTER TABLE invites ADD COLUMNS (new_col2 INT COMMENT 'a comment');
      hive> ALTER TABLE invites REPLACE COLUMNS (foo INT, bar STRING, baz INT COMMENT 'baz replaces new_col2');
  
 Note that REPLACE COLUMNS replaces all existing columns and only changes the table's schema, not the data. The table must use a native SerDe. REPLACE COLUMNS can also be used to drop columns from the table's schema:

  hive> ALTER TABLE invites REPLACE COLUMNS (foo INT COMMENT 'only keep the first column');
Dropping tables:

  hive> DROP TABLE pokes;
  
  
  DML Operations
  Hive Data Manipulation Language.
  
  Loading data from flat files into Hive:

  hive> LOAD DATA LOCAL INPATH './examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;
Loads a file that contains two columns separated by ctrl-a into pokes table. 'LOCAL' signifies that the input file is on the local file system. If 'LOCAL' is omitted then it looks for the file in HDFS.

The keyword 'OVERWRITE' signifies that existing data in the table is deleted. If the 'OVERWRITE' keyword is omitted, data files are appended to existing data sets.

NOTES:

NO verification of data against the schema is performed by the load command.
If the file is in hdfs, it is moved into the Hive-controlled file system namespace.
The root of the Hive directory is specified by the option hive.metastore.warehouse.dir in hive-default.xml. We advise users to create this directory before trying to create tables via Hive.
  hive> LOAD DATA LOCAL INPATH './examples/files/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
  hive> LOAD DATA LOCAL INPATH './examples/files/kv3.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-08');
The two LOAD statements above load data into two different partitions of the table invites. Table invites must be created as partitioned by the key ds for this to succeed.

  hive> LOAD DATA INPATH '/user/myname/kv2.txt' OVERWRITE INTO TABLE invites PARTITION (ds='2008-08-15');
The above command will load data from an HDFS file/directory to the table.
Note that loading data from HDFS will result in moving the file/directory. As a result, the operation is almost instantaneous.


  SQL Operations
Example Queries
Some example queries are shown below. They are available in build/dist/examples/queries.
More are available in the Hive sources at ql/src/test/queries/positive.

SELECTS and FILTERS
  hive> SELECT a.foo FROM invites a WHERE a.ds='2008-08-15';
selects column 'foo' from all rows of partition ds=2008-08-15 of the invites table. The results are not stored anywhere, but are displayed on the console.

Note that in all the examples that follow, INSERT (into a Hive table, local directory or HDFS directory) is optional.

  hive> INSERT OVERWRITE DIRECTORY '/tmp/hdfs_out' SELECT a.* FROM invites a WHERE a.ds='2008-08-15';
selects all rows from partition ds=2008-08-15 of the invites table into an HDFS directory. The result data is in files (depending on the number of mappers) in that directory.
NOTE: partition columns if any are selected by the use of *. They can also be specified in the projection clauses.

Partitioned tables must always have a partition selected in the WHERE clause of the statement.

  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/local_out' SELECT a.* FROM pokes a;
selects all rows from pokes table into a local directory.

  hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a;
  hive> INSERT OVERWRITE TABLE events SELECT a.* FROM profiles a WHERE a.key < 100;
  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/reg_3' SELECT a.* FROM events a;
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_4' select a.invites, a.pokes FROM profiles a;
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT COUNT(*) FROM invites a WHERE a.ds='2008-08-15';
  hive> INSERT OVERWRITE DIRECTORY '/tmp/reg_5' SELECT a.foo, a.bar FROM invites a;
  hive> INSERT OVERWRITE LOCAL DIRECTORY '/tmp/sum' SELECT SUM(a.pc) FROM pc1 a;
selects the sum of a column. The avg, min, or max can also be used. Note that for versions of Hive which don't include HIVE-287, you'll need to use COUNT(1) in place of COUNT(*).

GROUP BY
  hive> FROM invites a INSERT OVERWRITE TABLE events SELECT a.bar, count(*) WHERE a.foo > 0 GROUP BY a.bar;
  hive> INSERT OVERWRITE TABLE events SELECT a.bar, count(*) FROM invites a WHERE a.foo > 0 GROUP BY a.bar;
Note that for versions of Hive which don't include HIVE-287, you'll need to use COUNT(1) in place of COUNT(*).

JOIN
  hive> FROM pokes t1 JOIN invites t2 ON (t1.bar = t2.bar) INSERT OVERWRITE TABLE events SELECT t1.bar, t1.foo, t2.foo;
MULTITABLE INSERT
  FROM src
  INSERT OVERWRITE TABLE dest1 SELECT src.* WHERE src.key < 100
  INSERT OVERWRITE TABLE dest2 SELECT src.key, src.value WHERE src.key >= 100 and src.key < 200
  INSERT OVERWRITE TABLE dest3 PARTITION(ds='2008-04-08', hr='12') SELECT src.key WHERE src.key >= 200 and src.key < 300
  INSERT OVERWRITE LOCAL DIRECTORY '/tmp/dest4.out' SELECT src.value WHERE src.key >= 300;
STREAMING
  hive> FROM invites a INSERT OVERWRITE TABLE events SELECT TRANSFORM(a.foo, a.bar) AS (oof, rab) USING '/bin/cat' WHERE a.ds > '2008-08-09';
This streams the data in the map phase through the script /bin/cat (like Hadoop streaming).
Similarly – streaming can be used on the reduce side (please see the Hive Tutorial for examples).





Compose is a tool for defining and running multi-container Docker applications.
With Compose, you use a Compose file to configure your application's services.
Then, using a single command, you create and start all the services
from your configuration. To learn more about all the features of Compose
see [the list of features](https://github.com/docker/docker.github.io/blob/master/compose/overview.md#features).

Compose is great for development, testing, and staging environments, as well as
CI workflows. You can learn more about each case in
[Common Use Cases](https://github.com/docker/docker.github.io/blob/master/compose/overview.md#common-use-cases).

Using Compose is basically a three-step process.

1. Define your app's environment with a `Dockerfile` so it can be
reproduced anywhere.
2. Define the services that make up your app in `docker-compose.yml` so
they can be run together in an isolated environment.
3. Lastly, run `docker-compose up` and Compose will start and run your entire app.

A `docker-compose.yml` looks like this:

    version: '2'

    services:
      web:
        build: .
        ports:
         - "5000:5000"
        volumes:
         - .:/code
      redis:
        image: redis

For more information about the Compose file, see the
[Compose file reference](https://github.com/docker/docker.github.io/blob/master/compose/compose-file/compose-versioning.md).

Compose has commands for managing the whole lifecycle of your application:

 * Start, stop and rebuild services
 * View the status of running services
 * Stream the log output of running services
 * Run a one-off command on a service

Installation and documentation
------------------------------

- Full documentation is available on [Docker's website](https://docs.docker.com/compose/).
- Code repository for Compose is on [GitHub](https://github.com/docker/compose).
- If you find any problems please fill out an [issue](https://github.com/docker/compose/issues/new/choose). Thank you!

Contributing
------------

[![Build Status](https://jenkins.dockerproject.org/buildStatus/icon?job=docker/compose/master)](https://jenkins.dockerproject.org/job/docker/job/compose/job/master/)

Want to help build Compose? Check out our [contributing documentation](https://github.com/docker/compose/blob/master/CONTRIBUTING.md).

Releasing
---------

Releases are built by maintainers, following an outline of the [release process](https://github.com/docker/compose/blob/master/project/RELEASE-PROCESS.md).
