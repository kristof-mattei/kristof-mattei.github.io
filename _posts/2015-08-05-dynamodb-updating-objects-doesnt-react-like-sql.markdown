---
author: Kristof
date: 2015-08-05 19:34:32+00:00
layout: post
title: 'DynamoDb & updating objects: it''s doesn''t react like SQL!'
categories:
- Programming
---

Today I stumbled upon the following bug:

We had an object with some properties that we wanted to update, but only if a certain property of that object is not set, i.e. it should be null.

```
{
    "Id": 1, // Id is the HashKey
}
```


In this case we wanted to update the object with Id 1, and set an attribute called `Foo` to `"Bar"`

To do this I wrote the following Javascript, using the [aws-sdk](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/index.html):

``` 
function updateObject(id) {
    var dynamodb = new AWS.DynamoDB();

    dynamodb.updateItem({ 
            Id: id 
        }, { 
            UpdateExpression: "SET Foo = :value", 
            ExpressionAttributeValues: {
                ":value": "Bar"
            },
            ConditionExpression: "attribute_not_exists(Foo)" 
        }, function(error, data) { 
            if(error) { 
                // TODO check that the error is a ConditionalCheckFailedException, in 
                // which case the Condition failed, otherwise something else might be off. 
                console.log("Error");
            } else {
                console.log("All good, we've updated the object");
            } 
        }
    );
}
```

Perfect!

Now assume we have have a range of 1 -> 12 in our table, where half of them already have the `Foo` attribute, so we should get 50% `Error`, and 50% `All good, ...` (which is the case).

However, what do we expect when we update an item with Id 13?

When I, in my mind, which talks (used to) talk SQL when thinging about a database, updating something that is not there, doesn't do anything.

Consider the following table:

```
CREATE TABLE Test(
    Id INT NOT NULL,
    Foo NVARCHAR(255) NULL
)
```


With the following query:

```
INSERT INTO Test (Id, Foo) VALUES (1, NULL), (2, N'Bar'), (3, NULL)
GO

--SELECT * FROM Test
--GO

UPDATE Test SET Foo = 'Bar' WHERE Id = 1 AND Foo IS NULL
IF @@ROWCOUNT = 1
BEGIN
    SELECT N'1 updated, set Foo to Bar'
END
ELSE
BEGIN
    SELECT N'1 not updated, Foo was already set'
END
GO

--SELECT * FROM Test
--GO

UPDATE Test SET Foo = 'Bar' WHERE Id = 2 AND Foo IS NULL
IF @@ROWCOUNT = 1
BEGIN
    SELECT N'2 updated, set Foo to Bar'
END
ELSE
BEGIN
    SELECT N'2 not updated, Foo was already set'
END

--SELECT * FROM Test
--GO
UPDATE Test SET Foo = 'Bar' WHERE Id = 7 AND Foo IS NULL -- 7 Doesn't exist!
IF @@ROWCOUNT = 1
BEGIN
    SELECT N'7 updated, set Foo to Bar'
END
ELSE
BEGIN
    SELECT N'7 not updated, because 7 doesn''t exist!'
END
```


This will print, along with some empty result sets, the following:

```
1 updated, set Foo to Bar
2 not updated, Foo was already set
7 not updated, because 7 doesn't exist!
```

Now, that knowledge in SQL doesn't apply to DynamoDb.

While testing on some non-existing values we saw that our code passed the testcases perfectly. That's not how it should be.

Let's take a look again at the [documentation](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#updateItem-property), this time do actually read the first line:


<blockquote>Edits an existing item's attributes, **or adds a new item to the table if it does not already exist**.</blockquote>

(emphasis mine).

{% include image.html name="homer-computer-doh.jpg" %}

So we need to guard ourselves against updates on non-existing items? How do we do that? Let's extend our `ConditionExpression`. Start by taking the original code, and change the `ConditionExpression` as highlighted:

```    
function updateObject(id) {
    var dynamodb = new AWS.DynamoDB();

    dynamodb.updateItem({ 
            Id: id 
        }, { 
            UpdateExpression: "SET Foo = :value", 
            ExpressionAttributeValues: {
                ":id": id,
                ":value": "Bar"
            },
            // make sure the object we're updating actually has
            // :id as Id, the side-effect of this is that if none of those
            // is found, it will throw a ConditionalCheckFailedException
            // which is what we want
            ConditionExpression: "Id = :id AND attribute_not_exists(Foo)" 
        }, function(error, data) { 
            if(error) { 
                // TODO check that the error is a ConditionalCheckFailedException, in 
                // which case the Condition failed, otherwise something else might be off. 
                console.log("Error");
            } else {
                console.log("All good, we've updated the object");
            } 
        }
    );
}
```
