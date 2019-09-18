# Manage performance for database instances Pt. 1
* Manage database workload in SQL Server
* Design and implement Elastic Scale for Azure SQL Database
* Select an appropriate service tier or edition
* Optimize database file and tempdb configuration

Pt. 2 includes...
* Optimize memory configuration
* Monitor and diagnose schedule and wait statistics using dynamic management objects
* Troubleshoot and analyze storage, IO, and cache issues
* Monitor Azure SQL Database query plans

## Manage database workload in SQL Server
SQL Server Resource Governor is a feature than you can use to manage SQL Server workload and system resource consumption. It enables you to specify limits on the amount of CPU, physical IO, and memory that incoming application requests can use.

### Several components managed by the Resource Governor...

* A **resource pool** defines the physical resources of the server and behaves much like 
a virtual server. SQL Server creates an internal pool and a default pool during installation, and
you can add user-defined resource pools. 
    - Each resource pool is configured with the following settings (except the external resource pool): 
        1. Minimum CPU%
        2. Maximum CPU%
        3. Minimum Memory %
        4. Maximum Memory %. 
        - The sum of Minimum CPU% and of Minimum Memory % for all resources pools cannot
    be more than 100. These values represent the guaranteed average amount of that resource
    that each resource pool can use to respond to requests. 
        - The Maximum CPU% and Maximum Memory % reflect the maximum average amount for the respective resources
| pool_id | name     | min_cpu_percent | max_cpu_percent | min_memory_percent | max_memory_percent | cap_cpu_percent | min_iops_per_volume | max_iops_per_volume |
|---------|----------|-----------------|-----------------|--------------------|--------------------|-----------------|---------------------|---------------------|
| 1       | internal | 0               | 100             | 0                  | 100                | 100             | 0                   | 0                   |
| 2       | default  | 0               | 100             | 0                  | 100                | 100             | 0                   | 0                   |        
    - _Internal_ SQL Server uses the internal resource pool for resources required to run
    the database engine. You cannot change the resource configuration for the internal
    resource pool. SQL Server creates one when you enable the Resource Governor.
    Default 
    - _External_ An external resource pool is a new type for SQL Server 2016 that was added
    to support R Services. You can add an external resource pool to allocate resources for other external processes. The configuration for an external resource pool differs from the other resource pool types and includes only the following settings: Maximum CPU%, Maximum Memory %, and Maximum Processes.
    - _User-defined resource pool_ You can add a resource pool to allocate resources for
    database operations related to a specific workload.
```
CREATE RESOURCE POOL poolExamBookDaytime
WITH (
 MIN_CPU_PERCENT = 50,
 MAX_CPU_PERCENT = 80,
 CAP_CPU_PERCENT = 90,
 AFFINITY SCHEDULER = (0 TO 3),
 MIN_MEMORY_PERCENT = 50,
 MAX_MEMORY_PERCENT = 100,
 MIN_IOPS_PER_VOLUME = 20,
 MAX_IOPS_PER_VOLUME = 100
);
GO
CREATE RESOURCE POOL poolExamBookNighttime
WITH (
 MIN_CPU_PERCENT = 0,
 MAX_CPU_PERCENT = 50,
 CAP_CPU_PERCENT = 50,
 AFFINITY SCHEDULER = (0 TO 3),
 MIN_MEMORY_PERCENT = 5,
 MAX_MEMORY_PERCENT = 15,
NOTE MAXIMUM NUMBER OF SUPPORTED RESOURCE POOLS PER INSTANCE
SQL Server supports a maximum of 64 resource pools per instance. 
328 CHAPTER 4 Optimize database objects and SQL infrastructure
 MIN_IOPS_PER_VOLUME = 45,
 MAX_IOPS_PER_VOLUME = 100
);
GO
ALTER RESOURCE GOVERNOR RECONFIGURE;
GO
```
* **Workload Groups**: serves as a container for session requests that have similar
classification criteria. A workload allows for aggregate monitoring of the sessions, and defines
policies for the sessions. Each workload group is in a resource pool, which represents a subset
of the physical resources of an instance of the Database Engine. When a session is started, the
Resource Governor classifier assigns the session to a specific workload group, and the session
must run using the policies assigned to the workload group and the resources defined for the
resource pool.
    - A user cannot change anything classified as an internal group, but can monitor it
    - User-defined workload groups can be moved from one resource pool to another.
    - https://docs.microsoft.com/en-us/sql/relational-databases/resource-governor/resource-governor-workload-group?view=sql-server-2017
* **Classifier User-Defined Functions**: As SQL Server receives a request from a session, the classification process assigns it to the workload group having matching characteristics. You can fine-tune the results of this process by creating classifier user-defined functions.
    - Let’s say that you want to establish a classification function to assign a request to a workload group based on the time of day. Furthermore, you want to use a lookup table for the
    start and end times applicable to a workload group. Note that you must create this table in the master database because Resource Governor uses schema bindings for classifier functions.
```
USE master
GO
CREATE TABLE tblClassificationTime (
 TimeOfDay SYSNAME NOT NULL,
 TimeStart TIME NOT NULL,
 TimeEnd TIME NOT NULL
) ;
GO
INSERT INTO tblClassificationTime
VALUES('apps', '8:00 AM', '6:00 PM');
GO
INSERT INTO tblClassificationTime
VALUES('reports', '6:00 PM', '8:00 AM');
GO
```
Next, you create the classifier function that uses the lookup table to instruct the Resource
Governor which workload group to use when classifying an incoming request. 
```
USE master;
GO
CREATE FUNCTION fnTimeOfDayClassifier()
RETURNS sysname
WITH SCHEMABINDING AS
BEGIN
 DECLARE @TimeOfDay sysname
 DECLARE @loginTime time
 SET @loginTime = CONVERT(time,GETDATE())
 SELECT
 TOP 1 @TimeOfDay = TimeOfDay
 FROM dbo.tblClassificationTime
 WHERE TimeStart <= @loginTime and TimeEnd >= @loginTime
 IF(@TimeOfDay IS NOT NULL)
 BEGIN
 RETURN @TimeOfDay 
330 CHAPTER 4 Optimize database objects and SQL infrastructure
 END
 RETURN N'default'
END;
GO
ALTER RESOURCE GOVERNOR with (CLASSIFIER_FUNCTION = dbo.fnTimeOfDayClassifier);
ALTER RESOURCE GOVERNOR RECONFIGURE;
GO 
```

* **Useful tables for monitoring resource consumption**
```
USE master

--Current runtime data
SELECT * FROM sys.dm_resource_governor_resource_pools;

SELECT * FROM sys.dm_resource_governor_workload_groups;

--Determine the workload group for each session
SELECT
 s.group_id,
 CAST(g.name as nvarchar(20)) AS WkGrp,
 s.session_id,
 s.login_time,
 CAST(s.host_name as nvarchar(20)) AS Host,
 CAST(s.program_name AS nvarchar(20)) AS Program
FROM sys.dm_exec_sessions s
INNER JOIN sys.dm_resource_governor_workload_groups g
 ON g.group_id = s.group_id
ORDER BY g.name ;  

SELECT
 r.group_id,
 g.name,
 r.status,
 r.session_id,
 r.request_id,
 r.start_time, 
 r.command,
 r.sql_handle,
 t.text
FROM sys.dm_exec_requests r
INNER JOIN sys.dm_resource_governor_workload_groups g
 ON g.group_id = r.group_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
 ORDER BY g.name 

-- Determine the classifier running the request 
SELECT
 s.group_id,
 g.name,
 s.session_id,
 s.login_time,
 s.host_name,
 s.program_name
FROM sys.dm_exec_sessions s
INNER JOIN sys.dm_resource_governor_workload_groups g
 ON g.group_id = s.group_id AND
 s.status = 'preconnect'
ORDER BY g.name; 

SELECT
 r.group_id,
 g.name,
 r.status,
 r.session_id,
 r.request_id,
 r.start_time,
 r.command,
 r.sql_handle,
 t.text
FROM sys.dm_exec_requests r
INNER JOIN sys.dm_resource_governor_workload_groups g
 ON g.group_id = r.group_id
 AND r.status = 'preconnect'
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
ORDER BY g.name; 
```


## Design and implement Elastic Scale for Azure SQL Database
**Elastic Scale** is a feature in SQL Database that you use to adjust the database capacity to match the scalability requirements for different applications. In other words, you can grow or shrink the database by using a technique known as **sharding**, which partitions your data across identically structured database. **Sharding** is useful when the application data in aggregate exceeds the maximum size supported by SQL Database or when you need to separate data by geography for compliance, latency, or geopolitical reasons. 

Elastic Scale provides an elastic database client library and a Split-Merge service that help simplify the management of your applications.

* **Elastic database client library** provides the following features:
    1. **Shard map management** _You first register each database as a_ **shard**, and then define a shard map manager that directs connection requests to the correct shard by using a sharding key or a key range. A sharding key is data such as a customer ID number that the database engine uses to keep related transactions in one database.
    2. **Data-dependent routing** Rather than define a connection in your application, you can use this feature to automatically assign a connection to the correct shard.
    3. **Multishard querying** The database engine uses this feature to process queries in parallel across separate shards and then combine the results into a single result set.
    4. **Shard elasticity** This feature monitors resource consumption for the current workload and dynamically allocates more resource as necessary and shrinks the database to its normal state when those resources are no longer required.
* **Split-Merge Service** allows you to add or remove databases as shards from the shard set
and redistribute the data across more or fewer shards. As demand increases, you can split the data out across a greater number of shards. Conversely, you can merge the data into fewer shards as demand lowers.
    - https://docs.microsoft.com/en-us/azure/sql-database/sql-database-elastic-scale-configure-deploy-split-and-merge


## Select and appropriate service tier or edition
Apparently, the exam tests us on this... more specifically, we should know the general features and limitations of each edition and the differences between each edition. 

- **Express** This edition is a free version of SQL Server with limited features that you can
use for small applications and Web sites. The maximum database size supported by
this edition is 10 GB. It uses up to 1 GB memory and to the lesser of 1 physical processor or 4 cores. There are three types of SQL Server 2016 Express to choose:
    - **LocalDB** You use LocalDB for a simple application with a local embedded database that runs in single-user mode.
    - **Express** You use Express when your application requires a small database only
    and does not require any other components packaged with SQL Server in the
    Standard edition. You can install this edition on a server and then enable remote
    connections to support multiple users.
    - **Express with Advanced Services** This edition includes the database engine as
    well as Full Text Search and Reporting Services.
- **Web** This edition is scalable up to 64 GB of memory and the lesser of 4 physical processors or 16 cores with a maximum database size of 524 PB. It includes the database
engine, but without support for availability groups and other high-availability features.
It also does not include many of the advanced security and replication features available in Standard or Enterprise edition, nor does it include the business intelligence
components such as Analysis Services and Reporting Services, among others. Web edition is intended for use only by Web hosters and third-party software service providers.
- **Standard** This edition scales to 128 GB of memory and the lesser of 4 physical processors or 24 cores. The maximum database size with Standard edition is 524 PB. This
edition includes core database and business intelligence functionality and includes basic high-availability and disaster recovery features, new security features such as rowlevel security and dynamic data masking, and access to non-relational data sources by
using JSON and PolyBase.
- **Enterprise** This edition includes all features available in the SQL Server platform and
provides the highest scalability, greatest number of security features, and the most
advanced business intelligence and analytics features. Like Standard edition, Enterprise
edition supports a database size up to 524 PB, but its only limits on memory and processor sizes are the maximums set by your operating system.
    - To support higher availability, this edition supports up to 8 secondary replicas, with up
    to two synchronous secondary replicas, in an availability group, online page and file restore,online indexing, fast recovery, mirrored backups, and the ability to hot add memory and CPU. 
    - For greater performance, Enterprise edition supports in-memory OLTP, table and index
    partitioning, data compression, Resource Governor, parallelism for partitioned tables,
    multiple file stream containers, and delayed durability, among other features.
    - Enterprise edition includes many security features not found in Standard edition. In
    particular, Always Encrypted protects data at rest and in motion. Additional security
    features exclusive to Enterprise edition include more finely-grained auditing, transparent data encryption, and extensible key management.
    - Features supporting data warehouse operations found only in Enterprise edition include change data capture, star join query optimizations, and parallel query processing
    on partitioned indexes and tables.
- **Developer** This edition is for developers that create, test, and demonstrate applications using any of the data platform components available in Enterprise edition. However, the Developer edition cannot be used in a production environment.
- **Evaluation** This edition is a free trial version of SQL Server 2016. You can use this for
up to 180 days that you can use to explore all of the features available in Enterprise
Edition before making a purchasing decision.

After you choose your type, you also have to choose a service tier. The service tier sets the maximum database size while the performance level determines the amount of CPU, memory, and IO thresholds which collectively are measured as a DTU.
- **Basic** This service tier has a maximum database size of 2 GB and performance level
of 5 DTUs. You use this option when you need a small database for an application or
website with relatively few concurrent requests. The benchmark transaction rate is
16,600 transactions per hour.
- **Standard** This service tier has a maximum database size of 250 GB and performance
levels ranging from 10 to 100 DTUs. You can use this option when a database needs
to support multiple applications and multiple concurrent requests for workgroup and
web applications. With 50 DTUs, the benchmark transaction rate is 2,570 transactions
per minute.
- **Premium** This service tier has a maximum database size of 1 TB and performance
levels ranging from 125 to 4,000 DTUs. You use this option for enterprise-level database requirements. With 1,000 DTUs, the benchmark transaction rate is 735 transactions per second.


## Optimize database file and tempdb configuration
One option for improving the performance of read and write operations is
to optimize the configuration of files that SQL Server uses to store data and log files. The
optimization goal is to reduce contention for storage and IO of files used not only by the
database, but also by tempdb.

* **Database file optimization**
    - File Optimization: Data and log files should be placed on separate physical disks for better performance. 
        - SQL Server writes each transaction to the log before updating the data file. By separating the two file types, the read/write head for the disk with the log file can work more efficiently without frequent interruptions by the writes to the data file
        - By default, the data and log files for a new database are placed on the same drive, and normally in the same directory as the system databases: 
        \Program Files\Microsoft SQL Server\MSSQL13.MSSQLSERVER\MSSQL\DATA. 
        You can move the data and log files to a new location by using the ALTER DATABASE command. Separating data and log files on separate drives is beneficial because it mitigates a potential failure. If the drive containing the data files fails, you can still access the log file from the other disk and recover data up to the point of failure. 
```
ALTER DATABASE PR_ENT_PROD_COPY
SET OFFLINE;
GO
ALTER DATABASE PR_ENT_PROD_COPY
MODIFY FILE (NAME = PR_ENT_PROD_COPY_Data, FILENAME = “<drive:\filepath>\
Data\PR_ENT_PROD_COPY_Data.mdf”);
GO
ALTER DATABASE PR_ENT_PROD_COPY
MODIFY FILE (NAME = PR_ENT_PROD_COPY_Log, FILENAME = “drive:filepath>\
Log\PR_ENT_PROD_COPY_Log.mdf”);
ALTER DATABASE PR_ENT_PROD_COPY
SET ONLINE;
```
* **File groups and secondary data files**
    - Every database has a primary filegroup. This filegroup contains the primary data file and any secondary files that are not put into other filegroups. User-defined filegroups can be created to group data files together for administrative, data allocation, and placement purposes.
    - For example, three files, Data1.ndf, Data2.ndf, and Data3.ndf, can be created on three disk drives, respectively, and assigned to the filegroup fgroup1. A table can then be created specifically on the filegroup fgroup1. Queries for data from the table will be spread across the three disks; this will improve performance. The same performance improvement can be accomplished by using a single file created on a RAID (redundant array of independent disks) stripe set. However, files and filegroups let you easily add new files to new disks.
    - Partitioning You can use partitioning to place a table across multiple filegroups    Each partition should be in its own filegroup to improve performance. To map a value
    in a partition function to a specific filegroup, you use a partition scheme. Let’s say
    you have four filegroups—FGYear1, FGYear2, FGYear3, and FGYear4—and a partition
    function PFYearRange that defines four partitions for a table. You can create a partition
    schema to apply the partition function to these filegroups...
```
CREATE PARTITION SCHEME PSYear
 AS PARTITION PFYearRange
 TO (FGYear1, FGYear2, FGYear3, FGYear4);     
```

### tempdb optimization
Because so many operations, such as cursors, temp tables, and sorts, to name a few, rely on tempdb, configuring tempdb properly is critical to the performance of the database engine. Consider performing the following steps to optimize tempdb configuration:
- **SIMPLE recovery model** By using the SIMPLE recovery model, which is the default, SQL Server reclaims log space automatically so that the space required for the database is kept as low as possible.
    - **SIMPLE Recovery Model:** Every transaction is still written to the transaction log, but once the transaction is complete and the data has been written to the data file the space that was used in the transaction log file is now re-usable by new transactions.  Since this space is reused, you can't do a point in time recovery, so the most recent restore point will either be the complete backup or the latest differential backup that was completed.  Also, since the space in the transaction log can be reused, the transaction log will not grow forever. "Full" recovery model is where it grows forever.
- **Autogrowth** You should keep the default setting which allows tempdb files to automatically grow as needed.
- **File placement** The tempdb data and log files should be placed on different disks than your production database data and log files. Do not place the tempdb data files on the C drive to prevent the server from failing to start after running out of hard drive space. In addition, be sure to place the tempdb log file on its own disk. Regardless, put tempdb files on fast drives.
- **Files per core** In general, the number of data files for tempdb should be a 1:1 ratio of data files to CPU cores. In SQL Server 2016, the setup wizard now assigns the correct number based on the number of logical processors that it detects on your server, up to a maximum of 8.
- **File size** When you configure the database engine at setup, the default file size recommended by the setup wizard of 8 MB for an initial size with an autogrowth setting of 64MB is conservative and too small for most implementations. Instead, consider starting with an initial size of 4,096 MB with an autogrowth setting of 512 MB to reduce contention and minimize the impact of uncontrolled tempdb growth on performance. If you dedicate a drive to tempdb, you can set up the log files evenly on the drive to avoid performance issues caused by SQL Server pausing user activity as it grows the log files.
