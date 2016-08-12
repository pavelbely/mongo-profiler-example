# Choosing optimal MongoDB index for a specific query

As you may known having an appropriate index can significantly improve query performance.
But what counts as appropriate index? I would like to share with you my experience of adding one.

We will profile the database of **Comrade** - food ordering app for restaurant of that name.

## Requirements
- MongoDB installed
- JRE TODO - which version

## Comrade application

Comrade is a RESTful web service build with Spring that uses MongoDB as data storage for customer orders. The orders are saved in a collection with the following schema:
```js
db.order.findOne()

{
    "customer" : "Natasha_Ivanova",
    "items" : [
        {
            "product" : "Stewed cabbage in a pot",
            "price" : 4.4,
            "quantity" : 1
        },
        {
            "product" : "A loaf of bread",
            "price" : 1.3,
            "quantity" : 3
        },
        {
            "product" : "Mojito",
            "price" : 3.9,
            "quantity" : 1
        }
    ],
    "bonusPoint" : false,
    "givenAsBonus" : false
}
```

One would argue that saving orders in SQL database is a better idea and probably it is. Ok, we're doing so just for demostration purposes.

Customers make orders by sending **POST** http requests to **Comrade** app with the body of that kind.
```js
POST /order HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
    "customer" : "Natasha_Ivanova",
    "items" : [
        {
            "product" : "Cold beetrot soup",
            "price" : 1.5,
            "quantity" : 1
        },
        {
            "product" : "A loaf of bread",
            "price" : 0.05,
            "quantity" : 1
        },
        {
            "product" : "Buttermilk",
            "price" : 0.5,
            "quantity" : 1
        }]
}
```

This order is then being processed, and the bill is served as http response:
```js
{
  "orderId": "57adc478e6eb6c237791f5e1",
  "total": 5.05,
  "comment": "Bon appetite, comrade!",
  "bonusPointsCount": 1
}
```

Comrade restaurant runs bonus program - each 10th customer's meal is free.
If customer has enough bonus points they are being spent and her meal is served free:
```js
{
  "orderId": "57adc761e6eb6c237791f5e5",
  "total": 0,
  "comment": "Enjoy your FREE meal, comrade!",
  "bonusPointsCount": 0
}
```

## Profiling MongoDB indexes
### Import data

First let's load some data into you database.
Please download [comrade.json](comrade.json) and import it using the following command
```
mongoimport --db comrade --collection order --file comrade.json
```
10000 documents should have been imported into **order** collection.

### Start Comrade application
TODO

### Set MongoDB profiler on

Now we need to set the profiler on for all the queries.
Please refer to [MongoDB documentation](https://docs.mongodb.com/manual/tutorial/manage-the-database-profiler) for more info on profiling options.
```js
use comrade;
db.setProfilingLevel(2);
```

### Make an order and see what happens at backend
Make a new order by sending http request as described above. If you are using Postman - please find Postman collection with order request attached.

Now view profiler data:
```js
db.system.profile.find().sort({$natural:-1}).pretty();

/* 1 */
{
    "op" : "insert",
    "ns" : "comrade.order",
    "query" : {
        "insert" : "order",
        "ordered" : true,
        "documents" : [
            {
                "_id" : ObjectId("57addc24e6eb6c40dc1bc6f2"),
                "customer" : "Natasha_Ivanova",
                "items" : [
                    {
                        "product" : "Cold beetrot soup",
                        "price" : 1.5,
                        "quantity" : 1
                    },
                    {
                        "product" : "A loaf of bread",
                        "price" : 0.05,
                        "quantity" : 1
                    },
                    {
                        "product" : "Buttermilk",
                        "price" : 0.5,
                        "quantity" : 1
                    }
                ],
                "bonusPoint" : true,
                "givenAsBonus" : false
            }
        ]
    },
    "ninserted" : 1,
    "keyUpdates" : 0,
    "writeConflicts" : 0,
    "numYield" : 0,
    "locks" : {
        "Global" : {
            "acquireCount" : {
                "r" : NumberLong(1),
                "w" : NumberLong(1)
            }
        },
        "Database" : {
            "acquireCount" : {
                "w" : NumberLong(1)
            }
        },
        "Collection" : {
            "acquireCount" : {
                "w" : NumberLong(1)
            }
        }
    },
    "responseLength" : 40,
    "protocol" : "op_query",
    "millis" : 0,
    "execStats" : {},
    "ts" : ISODate("2016-08-12T14:24:36.459Z"),
    "client" : "127.0.0.1",
    "allUsers" : [],
    "user" : ""
}

/* 2 */
{
    "op" : "command",
    "ns" : "comrade.order",
    "command" : {
        "count" : "order",
        "query" : {
            "customer" : "Natasha_Ivanova",
            "bonusPoint" : true
        }
    },
    "keyUpdates" : 0,
    "writeConflicts" : 0,
    "numYield" : 78,
    "locks" : {
        "Global" : {
            "acquireCount" : {
                "r" : NumberLong(158)
            }
        },
        "Database" : {
            "acquireCount" : {
                "r" : NumberLong(79)
            }
        },
        "Collection" : {
            "acquireCount" : {
                "r" : NumberLong(79)
            }
        }
    },
    "responseLength" : 62,
    "protocol" : "op_query",
    "millis" : 7,
    "execStats" : {},
    "ts" : ISODate("2016-08-12T14:24:36.459Z"),
    "client" : "127.0.0.1",
    "allUsers" : [],
    "user" : ""
}
```

These queries were executed by **Comrade** when processing your order.
As you may guess the second query was executed when searching for customers bonus points.
```js
...
"query" : {
    "customer" : "Natasha_Ivanova",
    "bonusPoint" : true
}
...
```
And the first one when saving your processed order to db.

### Using explain plan to measure queries performance

Ok, let's check how effective these queries are using **MongoDB explain plan**.

```js
db.order.find({ "customer" : "Natasha_Ivanova", "bonusPoint" : true }).explain("executionStats");

{
    "queryPlanner" : {
      ...
      "winningPlan" : {
            "stage" : "COLLSCAN",
            "filter" : {
                "$and" : [
                    {
                        "bonusPoint" : {
                            "$eq" : true
                        }
                    },
                    {
                        "customer" : {
                            "$eq" : "Natasha_Ivanova"
                        }
                    }
                ]
            },
            "direction" : "forward"
        }
    ...
    "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 1,
        "executionTimeMillis" : 14,
        "totalKeysExamined" : 0,
        "totalDocsExamined" : 10001,
    ...
}
```

The output is pretty long but for now we are only concerned about **winningPlan** being **COLLSCAN** (which is obvious since we haven't added any indexes yet) and **executionTimeMillis**=14 and **totalDocsExamined**=10001.
That means the whole collection has been scanned in order to find Natasha's bonus points and that took 14 milliseconds.
This might seem like not that much, but let's try to improve it a bit.

### Add an index and check whether it improves performance

Let's add compound index on **customer** and **bonusPoint** fields.
```js
db.order.createIndex({customer : 1, bonusPoints : 1});
```

And now run that explain query from the previous section again.
```js
{
    "queryPlanner" : {
        "plannerVersion" : 1,
        "namespace" : "comrade.order",
        "indexFilterSet" : false,
        "parsedQuery" : {
            "$and" : [
                {
                    "bonusPoint" : {
                        "$eq" : true
                    }
                },
                {
                    "customer" : {
                        "$eq" : "Natasha_Ivanova"
                    }
                }
            ]
        },
        "winningPlan" : {
            "stage" : "FETCH",
            "filter" : {
                "bonusPoint" : {
                    "$eq" : true
                }
            },
            "inputStage" : {
                "stage" : "IXSCAN",
                "keyPattern" : {
                    "customer" : 1.0,
                    "bonusPoints" : 1.0
                },
                "indexName" : "customer_1_bonusPoints_1",
                "isMultiKey" : false,
                "isUnique" : false,
                "isSparse" : false,
                "isPartial" : false,
                "indexVersion" : 1,
                "direction" : "forward",
                "indexBounds" : {
                    "customer" : [
                        "[\"Natasha_Ivanova\", \"Natasha_Ivanova\"]"
                    ],
                    "bonusPoints" : [
                        "[MinKey, MaxKey]"
                    ]
                }
            }
        },
        "rejectedPlans" : []
    },
    "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 1,
        "executionTimeMillis" : 0,
        "totalKeysExamined" : 1,
        "totalDocsExamined" : 1,
        "executionStages" : {
            "stage" : "FETCH",
            "filter" : {
                "bonusPoint" : {
                    "$eq" : true
                }
            },
            "nReturned" : 1,
            "executionTimeMillisEstimate" : 0,
            "works" : 2,
            "advanced" : 1,
            "needTime" : 0,
            "needYield" : 0,
            "saveState" : 0,
            "restoreState" : 0,
            "isEOF" : 1,
            "invalidates" : 0,
            "docsExamined" : 1,
            "alreadyHasObj" : 0,
            "inputStage" : {
                "stage" : "IXSCAN",
                "nReturned" : 1,
                "executionTimeMillisEstimate" : 0,
                "works" : 2,
                "advanced" : 1,
                "needTime" : 0,
                "needYield" : 0,
                "saveState" : 0,
                "restoreState" : 0,
                "isEOF" : 1,
                "invalidates" : 0,
                "keyPattern" : {
                    "customer" : 1.0,
                    "bonusPoints" : 1.0
                },
                "indexName" : "customer_1_bonusPoints_1",
                "isMultiKey" : false,
                "isUnique" : false,
                "isSparse" : false,
                "isPartial" : false,
                "indexVersion" : 1,
                "direction" : "forward",
                "indexBounds" : {
                    "customer" : [
                        "[\"Natasha_Ivanova\", \"Natasha_Ivanova\"]"
                    ],
                    "bonusPoints" : [
                        "[MinKey, MaxKey]"
                    ]
                },
                "keysExamined" : 1,
                "dupsTested" : 0,
                "dupsDropped" : 0,
                "seenInvalidated" : 0
            }
        }
    },
    ...
```

As you can see this index makes a difference. Only 1 index key and one document were examined during this query and that took less that a 1 milliseconds.
```js
"executionTimeMillis" : 0,
"totalKeysExamined" : 1,
"totalDocsExamined" : 1,
```
