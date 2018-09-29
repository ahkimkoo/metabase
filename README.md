# Metabase mongodb native query problem
When you create a query that filters yesterday's data through the metabase ui, the actual generated statement is:
```
[{"$project":{"timestamp~~~day":{"$let":{"vars":{"field":"$timestamp"},"in":{"___date":{"$dateToString":{"format":"%Y-%m-%d","date":"$$field"}}}}},"_id":"$_id","level":"$level","message":"$message","meta":"$meta","timestamp~~~default":{"$let":{"vars":{"field":"$timestamp"},"in":"$$field"}}}},{"$match":{"timestamp~~~day":{"$eq":{"___date":"2018-09-28"}}}},{"$project":{"_id":"$_id","level":"$level","message":"$message","meta":"$meta","timestamp~~~default":"$timestamp~~~default"}},{"$limit":2000}]
```
dateToString means that all data will be calculated and compared, and the speed will be very slow. The entire process cannot use the index of timestamp.

The best way to filter the date is of course using mongodb's index. The query speed using gt, lt expression is very fast, especially when your data volume is large.
Usually we use the following sentences to filter the date：
```
db.user_actions.aggregate([{"$match":{"timestamp":{"$lt":new Date(new Date().setHours(0,0,0,0)),"$gte":new Date(new Date(new Date().getTime()-86400000).setHours(0,0,0,0))}}}]
```

Unfortunately, metabase does not support js expressions. The above statement will prompt an error：
```
Unrecognized token 'new': was expecting 'null', 'true', 'false' or NaN at [Source: java.io.StringReader@52471000; line: 1, column: 35]
```

So I modified one of the lines of code to let native query support js expressions.

frontend/src/metabase/lib/api.js, line 99, add "if(body)body = body.replace(/\$\{([^\}]+)}/ig,(m,x)=>{return eval(x);});":
```
_makeRequest(method, url, headers, body, data, options) {
    if(body)body = body.replace(/\$\{([^\}]+)}/ig,(m,x)=>{return eval(x);});//add this line
    return new Promise((resolve, reject) => {
```
So it supports js expressions (such as "${...}"), we can use mongodb's index to filter the date using a statement similar to the following:
```
db.user_actions.aggregate([{"$match":{"timestamp":{"$lt":ISODate("${new Date(new Date().setHours(0,0,0,0)).toISOString()}"),"$gte":ISODate("${new Date(new Date(new Date().getTime()-86400000).setHours(0,0,0,0)).toISOString()}")}}}])
```
