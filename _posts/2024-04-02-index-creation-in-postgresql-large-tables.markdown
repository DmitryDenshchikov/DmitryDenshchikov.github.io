---
layout: post
title: "Index Creation in PostgreSQL Large Tables: Checklist for Developers"
date: 2024-04-02 12:00:00 +0100
categories: jekyll update
---

# Introduction
During my career, I sometimes was faced with a situation where I had to create an index 
in a large table with millions of records. There are many pitfalls that you should be aware of before 
performing such operations, so I created this checklist of recommendations that will be helpful for 
those who have never done this before.

> Note: I’m pretty sure that the checklist can be extended since there are so many different 
> scenarios when it comes to creating indexes. My experience doesn’t cover everything, 
> so feel free to drop me a message with your suggestions! :)

# Checklist
- The creation of indexes should be performed separately from other DDL operations
- The creation of indexes should be performed separately from other DML operations
- The creation of indexes should be performed separately from service code changes 
(in case if the operation will be performed during Liquibase, Flyway, etc. migrations along with new code deployment)
- Create indexes [concurrently](https://www.postgresql.org/docs/current/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)
to avoid locking a table while it’s indexed against writes (use the `CONCURRENTLY` option)
- Ensure that new index creation WILL NOT BE performed multiple times in parallel
(e.g. because of Liquibase migrations in multiple pods/instances)
- Check that index creation will not cause performance degradation if you are 
going to create an index on frequently changed columns
- If a target table already contains some indexes, check that new index creation will 
not cause performance degradation because of the total number of indexes
- Review the time in which you are going to create an index. There is a good chance that 
there will be performance degradation while the index is being created. It depends on the table size, complexity, and frequency of write operations. Usually, it’s better to perform such operations during off-peak hours.
- Check if the new index will be used when you plan it to be used. The best way is to simulate production
data locally and review the [query plans](https://www.postgresql.org/docs/current/using-explain.html) 
(`EXPLAIN` and `EXPLAIN ANALYZE`) on whether the index is used.
- If you plan to create an index during Liquibase migrations, the migration will be blocked 
while the index is being created (even if you use `CONCURRENTLY`). Therefore, there is a possible 
pitfall: depending on your infrastructure setup, after some time, the pod/service can be identified as failed and then it can be deleted/recreated. This may leave your application in an inconsistent state.
- If you want to replace a partial index with a full one on the same column set, it’s better to 
create firstly the new index and only then drop the partial one. When dropping the partial index, 
again perform the operation [concurrently](https://www.postgresql.org/docs/current/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY) 
(use the `CONCURRENTLY` option).

# Points elaboration

#### _"The creation of indexes should be performed separately from other DDL operations"_
The main idea here is to avoid all possible problems related to inconsistency and degradation.
Index creation is usually performed outside of a transaction block (because we have to build it 
concurrently to avoid table locking) which means that if creation failed for some reason and other DDL 
operations were performed successfully, your database may be left in an inconsistent state.

#### _"The creation of indexes should be performed separately from other DML operations"_
Here the idea is similar to the one described above, but this time we’re dealing with DML changes.

#### _"The creation of indexes should be performed separately from service code changes"_
The goal is to avoid possible degradation caused by oversight. For example, you can 
accidentally along with index creation, deploy code that should use the index, but eventually it doesn’t because of the 
query structure, so you end up with performance degradation.

#### _"Create indexes concurrently to avoid locking a table while it’s indexed against writes"_
This point is self-explained :) We simply avoid potential performance degradation caused by blocked writes.

#### _"Ensure that new index creation WILL NOT BE performed multiple times in parallel"_
If, for some reason, index creation is performed multiple times in parallel, you could end up with a database 
in an inconsistent state. For example, you might have an index that doesn’t work properly.

#### _"Check that index creation will not cause performance degradation if you are going to create an index on frequently changed columns"_
The main pitfall here is that during the `UPDATE` operation, all indexes that include columns being modified will 
be updated to reflect the changes. So there is overhead, and you should check that it will not harm performance 
in your case (e.g. simulate production data and operations locally and measure the performance before and 
after index creation).

#### _"If a target table already contains some indexes, check that new index creation will not cause performance degradation because of the total number of indexes"_
The idea here is similar to the previous point. Every time a row in a table is added or removed, all indexes on the 
table are updated. If you have a lot of indexes, the write overhead may become considerable, and it may 
slow down `INSERT` and `DELETE` operations.

#### _"Review the time in which you are going to create an index"_
According to the [PostgreSQL docs](https://www.postgresql.org/docs/current/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY:~:text=Of%20course%2C%20the%20extra%20CPU%20and%20I/O%20load%20imposed%20by%20the%20index%20creation%20might%20slow%20other%20operations.) 
when you create an index concurrently, extra CPU and I/O load are involved, and it may slow down other operations, 
so if possible you should check locally (by simulating a production scenario) that it will not harm performance 
in your case. In case if it harms, it’s better to perform index creation during off-peak hours.

#### _"Check if the new index will be used when you plan it to be used"_
While it seems obvious, sometimes it can be forgotten due to paying attention to all other points :)
You can to play a bit on your local environment with queries, amount of data and the index that you are going to create 
and check that the index will be used and will be used efficiently.

#### _"If you plan to create an index during Liquibase migrations, the migration will be blocked while the index is being created"_
The main problem here is that a Liquibase migration with index creation blocks application startup completion 
(until the index is created and the corresponding migration is completed). You should keep this in mind because 
in that case, health checks for a service may produce failures, and the corresponding pod or instance can be 
removed and may leave your application in an inconsistent state.

#### _"If you want to replace a partial index with a full one on the same column set, it’s better to create firstly the new index and only then drop the partial one"_
The main idea here is to not leave your table without an index that is used. 
If you firstly create a full index on the same column set, the partial index will be used when it can be used and 
for other operations on the columns, the full index will be used. Yes, you will have two indexes for some 
time (until you drop the partial one) and there is overhead, but it’s relatively a small problem in comparison with 
the case when you firstly drop the partial index and leave your table without an index for some time (you will have 
a significant degradation)

