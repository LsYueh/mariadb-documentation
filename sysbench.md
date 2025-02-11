# Using Sysbench to Performance Test MariaDB

## Introduction

The idea behind a sysbench test is to test the current database performance with the given configuration which is currently set. This test does not test the application/application database performance but instead creates a dummy data set depending on the parameters we set. The test is run as a complex OLTP workload and tests the database throughput given a set concurrency for a specific amount of time. 

Once tested with the original confgiguration, we can then modify the MariaDB tuning paraemters and re run the same test on the same test data which was earlier generated by sysbench to get a proper representation of the results.

## Download sysbench

Download install the latest sysbench on the server where you want to generate load from. You may need to install `curl` on your system before this as it uses curl to setup the repositories.

#### Ubuntu

```txt
shell> curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
shell> sudo apt -y install sysbench
```

#### RHEL

```txt
shell> curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
shell> sudo yum -y install sysbench
```
or 

Download the latest package through https://repo.percona.com/yum/release/7/RPMS/x86_64/sysbench-tpcc-1.0.20-6.el7.x86_64.rpm

Once sysbench is installed, verify the version. It should be 1.0.17 or higher.

```txt
shell> sysbench --version
sysbench 1.0.17
```

Connect to your MariaDB server using the MariaDB CLI and create a database called `sbtest`

```txt
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5210
Server version: 10.4.8-MariaDB-1:10.4.8+maria~bionic mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database sbtest;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> create user sbuser@'%' identified by 'password';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> grant all on sbtest.* to sbuser@'%';
Query OK, 0 rows affected (0.002 sec)
```

#### Generate the Initial Data Volume

In this step, we will generate some tables and populate data into those tables under the `sbtest` database using `sbuser`. These are all configurable properties. 

The `sysbench` will use the following parameters:

- `--db-driver=mysql`
  - Use MySQL drivers which works for MariaDB as well
- `--threads=8`
  - The number of concurrent connections. This should be started with a lower and increased for each test to see the peak performance/limit of the database for the given hardware
- `--tables=12`
  - How many tables to create
- `--table-size=100000`
  - Number of rows to insert in each table for the loadtest
- `/usr/share/sysbench/oltp_common.lua`
  - You have to look for this file and make sure the path is correct after the installation, within the above path there are other test LUA files which you might want to use depending on your test scenarios.
- `--mysql-host=192.168.56.1 --mysql-port=3306 --mysql-user=sb_user --mysql-password=password`
  - These point to your MariaDB server or the MaxScale or directly to your database instance, if you have more than one MaxScale nodes, it can be a coma separated list for `--mysql-host` parameter
- `--mysql_db=sbtest`
  - Indicates which database to use for sysbench
- `prepare`
  - This indicates that we are preparing the envionment before the actual test. This will create those tables and pump the initial data to for the actual performance test

Executing sysbench with above mentioned parameters, we can expect to see the following results. By the end of this, we will have our tables creted and data populated based on our `table-size=100000` parameter, in this case, 100k rows in each table.

The following variabes and setup works with **sysbench 1.0.20**

```txt
shell> sysbench --db-driver=mysql --threads=8 --tables=20 --table_size=100000 /usr/share/sysbench/oltp_common.lua --mysql-host=192.168.56.1 --mysql-port=3306 --mysql-user=sbuser --mysql-password=P@ssw0rd --mysql_db=sbtest prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Creating table 'sbtest1'...
Inserting 100000 records into 'sbtest1'
Creating secondary indexes on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 100000 records into 'sbtest2'
Creating secondary indexes on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 100000 records into 'sbtest3'
Creating secondary indexes on 'sbtest3'...
Creating table 'sbtest4'...
Inserting 100000 records into 'sbtest4'
Creating secondary indexes on 'sbtest4'...
Creating table 'sbtest5'...
Inserting 100000 records into 'sbtest5'
Creating secondary indexes on 'sbtest5'...
Creating table 'sbtest6'...
Inserting 100000 records into 'sbtest6'
Creating secondary indexes on 'sbtest6'...
Creating table 'sbtest7'...
Inserting 100000 records into 'sbtest7'
Creating secondary indexes on 'sbtest7'...
Creating table 'sbtest8'...
Inserting 100000 records into 'sbtest8'
Creating secondary indexes on 'sbtest8'...
Creating table 'sbtest9'...
Inserting 100000 records into 'sbtest9'
Creating secondary indexes on 'sbtest9'...
Creating table 'sbtest10'...
Inserting 100000 records into 'sbtest10'
Creating secondary indexes on 'sbtest10'...
Creating table 'sbtest11'...
Inserting 100000 records into 'sbtest11'
Creating secondary indexes on 'sbtest11'...
Creating table 'sbtest12'...
Inserting 100000 records into 'sbtest12'
Creating secondary indexes on 'sbtest12'...
```

#### Execute the benchmark

Once the database is ready with tables and dummy data, we can now run the test.

The following is exactly the same as the previous, just the last parameter, instead of `prepare` we will use `run`

```txt
shell> sysbench --db-driver=mysql --threads=8 --tables=20 --table_size=100000 /usr/share/sysbench/oltp_read_write.lua --mysql-host=192.168.56.1 --mysql-port=3306 --mysql-user=sbuser --mysql-password=P@ssw0rd --mysql_db=sbtest --time=120 --report-interval=10 run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Report intermediate results every 10 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 10s ] thds: 8 tps: 803.21 qps: 16075.09 (r/w/o: 11254.60/3213.26/1607.23) lat (ms,95%): 15.00 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 8 tps: 825.98 qps: 16522.47 (r/w/o: 11564.17/3306.33/1651.97) lat (ms,95%): 14.46 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 8 tps: 884.87 qps: 17697.28 (r/w/o: 12388.57/3538.98/1769.74) lat (ms,95%): 13.95 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 8 tps: 825.28 qps: 16507.08 (r/w/o: 11554.71/3301.82/1650.56) lat (ms,95%): 14.21 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 8 tps: 811.71 qps: 16234.42 (r/w/o: 11363.96/3247.04/1623.42) lat (ms,95%): 14.73 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 8 tps: 819.49 qps: 16388.88 (r/w/o: 11472.35/3277.56/1638.98) lat (ms,95%): 13.95 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            696024
        write:                           198864
        other:                           99432
        total:                           994320
    transactions:                        49716  (828.45 per sec.)
    queries:                             994320 (16569.03 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0080s
    total number of events:              49716

Latency (ms):
         min:                                    3.96
         avg:                                    9.65
         max:                                   31.18
         95th percentile:                       14.46
         sum:                               479830.97

Threads fairness:
    events (avg/stddev):           6214.5000/16.76
    execution time (avg/stddev):   59.9789/0.00
```

The PREPARE is to be done only once, but the RUN can be performed multiple times, this will reuse the originally prepared data and generate a complex SELECT, INSERT, UPDATE, DELETE load while maintaining transactions. A transaction will have multiple statements executuins within but the total number of queries per second will be much higher.

Generally, after PREPARE, we can tune the server parameters like buffer_pool_size, tmp_table_size, encryption etc and rerun sysbench using `RUN` argument multiple times to see the impact of those parameters. 

For instance, the output above says `transactions:                        49716  (828.45 per sec.)` but the `queries:                             994320 (16569.03 per sec.)` is much higher since each transaction has many smaller events being generated.

Repeat the above and increase the threads to 8, 16, 32, 64, 128 and so on just to see when the performance stops to drop. That will be the peak capacity of your server.

`--time=60` will execute load test for 60 seconds, recommended to increase this number to test for 30 minutes or higher to test how long the server can maintain the performance, over time the server's performance should stablise.

#### Cleanup

Finally, once done, you can run the same command with `CLEANUP` parameter to destroy the test data generated for this test. Once cleanup is done, you will have to `PREPARE` once again before re-running the test.

```txt
shell> sysbench --db-driver=mysql --threads=8 --oltp-tables-count=12 --oltp-table-size=100000 --oltp-test-mode=complex --oltp-dist-type=uniform /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua --mysql-host=192.168.56.1 --mysql-port=3306 --mysql-user=sbuser --mysql-password=password --mysql_db=sbtest --time=60 --report-interval=10 cleanup 
sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)

Dropping table 'sbtest1'...
Dropping table 'sbtest2'...
Dropping table 'sbtest3'...
Dropping table 'sbtest4'...
Dropping table 'sbtest5'...
Dropping table 'sbtest6'...
Dropping table 'sbtest7'...
Dropping table 'sbtest8'...
Dropping table 'sbtest9'...
Dropping table 'sbtest10'...
Dropping table 'sbtest11'...
Dropping table 'sbtest12'...
```

Thank you.
