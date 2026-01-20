+++
title = "The impact of SqlDataReader.GetOrdinal on performance"
date = 2014-05-25 11:56:53+00:00

[taxonomies]
categories = [".NET", "C#", "MSSQL", "Programming"]

[extra]
author = "Kristof"
+++

I recently had a discussion about the impact of [`SqlDataReader.GetOrdinal`](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqldatareader.getordinal.aspx) on execution of a [`SqlClient.SqlCommand`](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlcommand.aspx). I then decided to run some code to measure the difference, because I think that's the only way to get a decent opinion. This is the code that I've used to run a certain query 1000 times:

```csharp
private void InvokeQuery(Action mapObject)
{
    Stopwatch stopwatch = Stopwatch.StartNew();

    for (int i = 0; i < Iterations; i++)
    {
        using (var sqlCommand = new SqlCommand(this._query, this._sqlConnection))
        {
            using (SqlDataReader sqlDataReader = sqlCommand.ExecuteReader())
            {
                while (sqlDataReader.NextResult())
                {
                    mapObject(sqlDataReader);
                }
            }
        }
    }

    stopwatch.Stop();

    Debug.WriteLine("Running {0} queries took {1} milliseconds!", Iterations, stopwatch.ElapsedMilliseconds);
}
```

`mapObject` uses either directly the ordinal, or fetches the ordinal based on the column name. Also, I moved everything inside of the `for` loop to ensure nothing could be reused between queries. Here are the `mapObject` [`Action`](http://msdn.microsoft.com/en-us/library/system.action.aspx)s, with `GetOrdinal`:

    Action<SqlDataReader> = sqlDataReader =>
    {
        int salesOrderID = sqlDataReader.GetOrdinal("SalesOrderID");
        int revisionNumber = sqlDataReader.GetOrdinal("RevisionNumber");
        int orderDate = sqlDataReader.GetOrdinal("OrderDate");
        int dueDate = sqlDataReader.GetOrdinal("DueDate");
        int shipDate = sqlDataReader.GetOrdinal("ShipDate");
        int status = sqlDataReader.GetOrdinal("Status");
        int onlineOrderFlag = sqlDataReader.GetOrdinal("OnlineOrderFlag");
        int salesOrderNumber = sqlDataReader.GetOrdinal("SalesOrderNumber");
        int purchaseOrderNumber = sqlDataReader.GetOrdinal("PurchaseOrderNumber");
        int accountNumber = sqlDataReader.GetOrdinal("AccountNumber");
        int customerID = sqlDataReader.GetOrdinal("CustomerID");
        int salesPersonID = sqlDataReader.GetOrdinal("SalesPersonID");
        int territoryID = sqlDataReader.GetOrdinal("TerritoryID");
        int billToAddressID = sqlDataReader.GetOrdinal("BillToAddressID");
        int shipToAddressID = sqlDataReader.GetOrdinal("ShipToAddressID");
        int shipMethodID = sqlDataReader.GetOrdinal("ShipMethodID");
        int creditCardID = sqlDataReader.GetOrdinal("CreditCardID");
        int creditCardApprovalCode = sqlDataReader.GetOrdinal("CreditCardApprovalCode");
        int currencyRateID = sqlDataReader.GetOrdinal("CurrencyRateID");
        int subTotal = sqlDataReader.GetOrdinal("SubTotal");
        int taxAmt = sqlDataReader.GetOrdinal("TaxAmt");
        int freight = sqlDataReader.GetOrdinal("Freight");
        int totalDue = sqlDataReader.GetOrdinal("TotalDue");
        int comment = sqlDataReader.GetOrdinal("Comment");
        int rowguid = sqlDataReader.GetOrdinal("rowguid");
        int modifiedDate = sqlDataReader.GetOrdinal("ModifiedDate");

        var temp = new SalesOrderHeader(
            salesOrderID: sqlDataReader.GetInt32(salesOrderID),
            revisionNumber: sqlDataReader.GetInt16(revisionNumber),
            orderDate: sqlDataReader.GetDateTime(orderDate),
            dueDate: sqlDataReader.GetDateTime(dueDate),
            shipDate: sqlDataReader.GetDateTime(shipDate),
            status: sqlDataReader.GetInt16(status),
            onlineOrderFlag: sqlDataReader.GetBoolean(onlineOrderFlag),
            salesOrderNumber: sqlDataReader.GetString(salesOrderNumber),
            purchaseOrderNumber: sqlDataReader.GetString(purchaseOrderNumber),
            accountNumber: sqlDataReader.GetString(accountNumber),
            customerID: sqlDataReader.GetInt32(customerID),
            salesPersonID: sqlDataReader.GetInt32(salesPersonID),
            territoryID: sqlDataReader.GetInt32(territoryID),
            billToAddressID: sqlDataReader.GetInt32(billToAddressID),
            shipToAddressID: sqlDataReader.GetInt32(shipToAddressID),
            shipMethodID: sqlDataReader.GetInt32(shipMethodID),
            creditCardID: sqlDataReader.GetInt32(creditCardID),
            creditCardApprovalCode: sqlDataReader.GetString(creditCardApprovalCode),
            currencyRateID: sqlDataReader.GetInt32(currencyRateID),
            subTotal: sqlDataReader.GetDecimal(subTotal),
            taxAmt: sqlDataReader.GetDecimal(taxAmt),
            freight: sqlDataReader.GetDecimal(freight),
            totalDue: sqlDataReader.GetDecimal(totalDue),
            comment: sqlDataReader.GetString(comment),
            rowguid: sqlDataReader.GetGuid(rowguid),
            modifiedDate: sqlDataReader.GetDateTime(modifiedDate)
            );
    };

And without `GetOrdinal`:

    Action<SqlDataReader> mapSalesOrderHeader = sqlDataReader =>
    {
        new SalesOrderHeader(
            salesOrderID: sqlDataReader.GetInt32(0),
            revisionNumber: sqlDataReader.GetInt16(1),
            orderDate: sqlDataReader.GetDateTime(2),
            dueDate: sqlDataReader.GetDateTime(3),
            shipDate: sqlDataReader.GetDateTime(4),
            status: sqlDataReader.GetInt16(5),
            onlineOrderFlag: sqlDataReader.GetBoolean(6),
            salesOrderNumber: sqlDataReader.GetString(7),
            purchaseOrderNumber: sqlDataReader.GetString(8),
            accountNumber: sqlDataReader.GetString(9),
            customerID: sqlDataReader.GetInt32(10),
            salesPersonID: sqlDataReader.GetInt32(11),
            territoryID: sqlDataReader.GetInt32(12),
            billToAddressID: sqlDataReader.GetInt32(13),
            shipToAddressID: sqlDataReader.GetInt32(14),
            shipMethodID: sqlDataReader.GetInt32(15),
            creditCardID: sqlDataReader.GetInt32(16),
            creditCardApprovalCode: sqlDataReader.GetString(17),
            currencyRateID: sqlDataReader.GetInt32(18),
            subTotal: sqlDataReader.GetDecimal(19),
            taxAmt: sqlDataReader.GetDecimal(20),
            freight: sqlDataReader.GetDecimal(21),
            totalDue: sqlDataReader.GetDecimal(22),
            comment: sqlDataReader.GetString(23),
            rowguid: sqlDataReader.GetGuid(24),
            modifiedDate: sqlDataReader.GetDateTime(25));
    };

With `GetOrdinal` the results are:

{{ image(name="CreateWithGetOrdinal.png") }}

And without:

{{ image(name="CreateWithoutGetOrdinal.png") }}

As you can see the performance difference is so low that I honestly don't think you should sacrifice the readability and maintainability of your code vs a mere 82 milliseconds on a 1000 queries. Readability speaks for itself, you don't talk with `int`s anymore, and for maintainability, consider the following: If your query column(s) change and you forget to update your code, `GetOrdinal` will throw an [IndexOutOfRangeException](http://msdn.microsoft.com/en-us/library/system.indexoutofrangeexception.aspx), instead of **maybe** get an [InvalidCastException](http://msdn.microsoft.com/en-us/library/system.invalidcastexception.aspx) or, if you're really unlucky, another column and then broken code behavior... One [sidenote](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqldatareader.getordinal.aspx) to add:

> `GetOrdinal` performs a case-sensitive lookup first. If it fails, a second, case-insensitive search occurs (a case-insensitive comparison is done using the database collation). Unexpected results can occur when comparisons are affected by culture-specific casing rules. For example, in Turkish, the following example yields the wrong results because the file system in Turkish does not use linguistic casing rules for the letter 'i' in "file". The method throws an `IndexOutOfRange` exception if the zero-based column ordinal is not found.

So do watch out that case.

PS: the project itself is hosted on GitHub, you can find it [here](https://github.com/kristof-mattei/get-ordinal-or-not)!
