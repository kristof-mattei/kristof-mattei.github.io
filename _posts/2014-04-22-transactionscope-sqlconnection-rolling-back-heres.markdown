---
author: Kristof
comments: true
date: 2014-04-22 16:39:29+00:00
layout: post
slug: transactionscope-sqlconnection-rolling-back-heres
title: TransactionScope & SqlConnection not rolling back? Here's why...
wordpress_id: 2120
categories:
- .NET
- C#
- MSSQL
- Programming
---

A while back we ran into an issue with one of our projects where we executed a erroneous query (missing DELETE statement), and then left the database in an inconsistent state.

Which is weird, considering the fact that we use a [`TransactionScope`](http://msdn.microsoft.com/en-us/library/system.transactions.transactionscope.aspx).

After some digging around I found the behavior I wanted, and how to write it in correct C#.

Allow me to elaborate.

Consider a database with 3 tables:

```
T2 --> T1 <-- T3
```

Where both `T2` and `T3` link to an entity in `T1`, thus we cannot delete lines from `T1` that are still referenced in `T2` or `T3`.

I jumped to C# and started playing with some code, and discovered the following (mind you, each piece of code is actually supposed to throw an exception and abort):

This doesn't use a `TransactionScope`, thus leaving the database in an inconsistent state:

```csharp
using (var sqlConnection = new SqlConnection(ConnectionString))
{
    sqlConnection.Open();

    using (SqlCommand sqlCommand = sqlConnection.CreateCommand())
    {
        sqlCommand.CommandText = "USE [TransactionScopeTests]; DELETE FROM T3; DELETE FROM T1;"; 
        // DELETE FROM T1 will cause violation of integrity, because rows from T2 are still using rows from T1.

        sqlCommand.ExecuteNonQuery();
    } 
}
```

Now I wanted to wrap this in a TransactionScope, so I tried this:

```csharp
using (var sqlConnection = new SqlConnection(ConnectionString))
{
    sqlConnection.Open();

    using (var transactionScope = new TransactionScope())
    {
        using (SqlCommand sqlCommand = sqlConnection.CreateCommand())
        {
            sqlCommand.CommandText = "USE [TransactionScopeTests]; DELETE FROM T3; DELETE FROM T1;"; 

            sqlCommand.ExecuteNonQuery();
        }

        transactionScope.Complete();
    }
}
```

Well guess what, this essentially fixes nothing. The database, upon completion of the `ExecuteNonQuery()` is left in the same inconsistent state. `T3` was empty, which shouldn't happen since the delete from `T1` failed.

So what is the correct behavior?

Well, it doesn't matter whether you create the `TransactionScope` or the `SqlConnection` first, **as long as you `Open()` the `SqlConnection` inside of the `TransactionScope`**:

```csharp
using (var transactionScope = new TransactionScope())
{
    using (var sqlConnection = new SqlConnection(ConnectionString))
    {
        sqlConnection.Open();

        using (SqlCommand sqlCommand = sqlConnection.CreateCommand())
        {
            sqlCommand.CommandText = "USE [TransactionScopeTests]; DELETE FROM T3; DELETE FROM T1;"; 

            sqlCommand.ExecuteNonQuery();
        }

        transactionScope.Complete();
    }
}                                                                                                                           
```


Or the inverse (swapping the declaration of the `TransactionScope` and `SqlConnection`):

```csharp
using (var sqlConnection = new SqlConnection(ConnectionString))
{
    using (var transactionScope = new TransactionScope())
    {
        sqlConnection.Open();

        using (SqlCommand sqlCommand = sqlConnection.CreateCommand())
        {
            sqlCommand.CommandText = "USE [TransactionScopeTests]; DELETE FROM T3; DELETE FROM T1;"; 

            sqlCommand.ExecuteNonQuery();
        }

        transactionScope.Complete();
    }
}
```

I wrote the test cases on a project on GitHub which you can download, compile and run as Tests for yourself!

[https://github.com/kristof-mattei/transaction-scope](https://github.com/kristof-mattei/transaction-scope)

Have a good one,

-Kristof
