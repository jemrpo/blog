---
title: "Updating Large Tables in Azure SQL Database, Lessons Learned"
date: 2024-01-15T13:24:08-05:00
tags: ["Azure", "SQL", "Databases", "SQL Server", "Azure SQL", "Data Engineering", "ETL"]
toc: true
---

## The Problem
Sometimes you have some large tables in an Azure SQL Database with around two hundred million rows (200.000.000). You need to filter the rows within a column and update the rows with ``NULL`` values. You think a single query will do it, so you run the following:
```sql
UPDATE dbo.myTable
SET last_updated = '2023-12-31 00:00:00.000'
WHERE last_updated IS NULL
AND [Date] < '2024-01-01'
```

But it's taking a long time, and when you think the task is almost complete, it fails... :confused:
You get an error message like this:
```log
The session has been terminated because of excessive transaction log space usage.
Try modifying fewer rows in a single transaction.
```

Now, you feel like pretty much wasted 12 hours of your time. :tired_face:\
At least you know what is breaking it, The transaction log is filling up... :pensive:\
But still, you need to get this done fast. :worried:


## Why Is This Happening?
The problem is that you are trying to update a large table, and you are doing it in a single transaction. This means that SQL will keep a copy of the original data in the transaction log, and it will keep it there until the transaction is complete. This is a problem because the transaction log is limited in size, and it will fill up. When this happens, SQL Server will terminate the transaction, and you will get the error message above.

The first thing most of us would think is that the transaction log needs to be shrunk, or its size needs to be increased;
To get the size and the utilization of the database and the transaction log you run the following query:
```sql
SELECT file_id, type_desc,
       CAST(FILEPROPERTY(name, 'SpaceUsed') AS decimal(19,4)) * 8 / 1024. AS space_used_mb,
       CAST(size/128.0 - CAST(FILEPROPERTY(name, 'SpaceUsed') AS int)/128.0 AS decimal(19,4)) AS space_unused_mb,
       CAST(size AS decimal(19,4)) * 8 / 1024. AS space_allocated_mb,
       CAST(max_size AS decimal(19,4)) * 8 / 1024. AS max_size_mb
FROM sys.database_files;
```

And then you get something like this:
```log
| file_id | type_desc | space_used_mb   | space_unused_mb | space_allocated_mb | max_size_mb      |
|--------|------------|-----------------|-----------------|--------------------|------------------|
| 2      | LOG        | 57.445312500    | 44622.5547      | 44680.000000000    | 184320.000000000 |
| 3      | ROWS       | 22109.312500000 | 43426.6875      | 65536.000000000    | 65536.000000000  |
| 65537  | FILESTREAM | NULL            | NULL            | 0.000000000        | -0.007812500     |
```

Now, you can see; that the transaction log has plenty of unused space, so shrinking or making it bigger won't help. After a little bit of research, you find that Azure SQL Database automatically [shrinks the transaction log](https://learn.microsoft.com/en-us/azure/azure-sql/database/file-space-manage?view=azuresql-db#shrink-transaction-log-file), therefore this is something you don't need to worry about most of the time.


## The Solution
The issue can occur in any DML operation such as insert, update, or delete. so it is recommended to review 
the transaction to avoid unnecessary writes. So one needs to reduce the number of rows that are operated on immediately by implementing batching or splitting into multiple smaller transactions. 
This has proven to be one of the most effective ways to perform this kind of operation, as described in the [documentation](https://learn.microsoft.com/en-us/azure/azure-sql/performance-improve-use-batching?view=azuresql-db&preserve-view=true).

Then, since you know that each day in my table has around 300.000 rows, you decide it'd be a good idea to perform 
the ``UPDATE`` operation based on the date. So, you come up with the following script:

```sql
DECLARE @TopDate DATE = '2023-12-31';
DECLARE @BottomDate DATE = '2021-01-01'; 

WHILE @TopDate >= @BottomDate
BEGIN

    UPDATE dbo.myTable
    SET last_updated = '2023-12-31 00:00:00.000'
    WHERE [Date] = @TopDate 
        AND last_updated IS NULL;

    SET @TopDate = DATEADD(DAY, -1, @TopDate);
END;
```

This script will update the rows in the table in batches of one day, starting from the top date and going down to the
bottom date. This way, the transaction log will not fill up, the changes will be reflected faster on the table, and 
the operation will be completed successfully.

This operation can take a long time to complete, it took me around 9 hours per table. It'd be a good idea to run it in through Apache Airflow, Azure Data Factory, AWS Glue, or any other (data) orchestration tool.
