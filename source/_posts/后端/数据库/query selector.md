---
title: mongoose query selector
categories: 
- 数据库
tags:
- 数据库
- mongodb
- mongoose
---

## Query Selectors

### Comparison

For comparison of different BSON type values, see the [specified BSON comparison order](https://docs.mongodb.com/manual/reference/bson-type-comparison-order/#std-label-bson-types-comparison-order).<!--more-->

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$eq`](https://docs.mongodb.com/manual/reference/operator/query/eq/#mongodb-query-op.-eq) | Matches values that are equal to a specified value.          |
| [`$gt`](https://docs.mongodb.com/manual/reference/operator/query/gt/#mongodb-query-op.-gt) | Matches values that are greater than a specified value.      |
| [`$gte`](https://docs.mongodb.com/manual/reference/operator/query/gte/#mongodb-query-op.-gte) | Matches values that are greater than or equal to a specified value. |
| [`$in`](https://docs.mongodb.com/manual/reference/operator/query/in/#mongodb-query-op.-in) | Matches any of the values specified in an array.             |
| [`$lt`](https://docs.mongodb.com/manual/reference/operator/query/lt/#mongodb-query-op.-lt) | Matches values that are less than a specified value.         |
| [`$lte`](https://docs.mongodb.com/manual/reference/operator/query/lte/#mongodb-query-op.-lte) | Matches values that are less than or equal to a specified value. |
| [`$ne`](https://docs.mongodb.com/manual/reference/operator/query/ne/#mongodb-query-op.-ne) | Matches all values that are not equal to a specified value.  |
| [`$nin`](https://docs.mongodb.com/manual/reference/operator/query/nin/#mongodb-query-op.-nin) | Matches none of the values specified in an array.            |

### Logical

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$and`](https://docs.mongodb.com/manual/reference/operator/query/and/#mongodb-query-op.-and) | Joins query clauses with a logical `AND` returns all documents that match the conditions of both clauses. |
| [`$not`](https://docs.mongodb.com/manual/reference/operator/query/not/#mongodb-query-op.-not) | Inverts the effect of a query expression and returns documents that do *not* match the query expression. |
| [`$nor`](https://docs.mongodb.com/manual/reference/operator/query/nor/#mongodb-query-op.-nor) | Joins query clauses with a logical `NOR` returns all documents that fail to match both clauses. |
| [`$or`](https://docs.mongodb.com/manual/reference/operator/query/or/#mongodb-query-op.-or) | Joins query clauses with a logical `OR` returns all documents that match the conditions of either clause. |

### Element

| Name                                                         | Description                                            |
| :----------------------------------------------------------- | :----------------------------------------------------- |
| [`$exists`](https://docs.mongodb.com/manual/reference/operator/query/exists/#mongodb-query-op.-exists) | Matches documents that have the specified field.       |
| [`$type`](https://docs.mongodb.com/manual/reference/operator/query/type/#mongodb-query-op.-type) | Selects documents if a field is of the specified type. |

### Evaluation

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$expr`](https://docs.mongodb.com/manual/reference/operator/query/expr/#mongodb-query-op.-expr) | Allows use of aggregation expressions within the query language. |
| [`$jsonSchema`](https://docs.mongodb.com/manual/reference/operator/query/jsonSchema/#mongodb-query-op.-jsonSchema) | Validate documents against the given JSON Schema.            |
| [`$mod`](https://docs.mongodb.com/manual/reference/operator/query/mod/#mongodb-query-op.-mod) | Performs a modulo operation on the value of a field and selects documents with a specified result. |
| [`$regex`](https://docs.mongodb.com/manual/reference/operator/query/regex/#mongodb-query-op.-regex) | Selects documents where values match a specified regular expression. |
| [`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#mongodb-query-op.-text) | Performs text search.                                        |
| [`$where`](https://docs.mongodb.com/manual/reference/operator/query/where/#mongodb-query-op.-where) | Matches documents that satisfy a JavaScript expression.      |

### Geospatial

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$geoIntersects`](https://docs.mongodb.com/manual/reference/operator/query/geoIntersects/#mongodb-query-op.-geoIntersects) | Selects geometries that intersect with a [GeoJSON](https://docs.mongodb.com/manual/reference/glossary/#std-term-GeoJSON) geometry. The [2dsphere](https://docs.mongodb.com/manual/core/2dsphere/) index supports [`$geoIntersects`](https://docs.mongodb.com/manual/reference/operator/query/geoIntersects/#mongodb-query-op.-geoIntersects). |
| [`$geoWithin`](https://docs.mongodb.com/manual/reference/operator/query/geoWithin/#mongodb-query-op.-geoWithin) | Selects geometries within a bounding [GeoJSON geometry](https://docs.mongodb.com/manual/reference/geojson/#std-label-geospatial-indexes-store-geojson). The [2dsphere](https://docs.mongodb.com/manual/core/2dsphere/) and [2d](https://docs.mongodb.com/manual/core/2d/) indexes support [`$geoWithin`](https://docs.mongodb.com/manual/reference/operator/query/geoWithin/#mongodb-query-op.-geoWithin). |
| [`$near`](https://docs.mongodb.com/manual/reference/operator/query/near/#mongodb-query-op.-near) | Returns geospatial objects in proximity to a point. Requires a geospatial index. The [2dsphere](https://docs.mongodb.com/manual/core/2dsphere/) and [2d](https://docs.mongodb.com/manual/core/2d/) indexes support [`$near`](https://docs.mongodb.com/manual/reference/operator/query/near/#mongodb-query-op.-near). |
| [`$nearSphere`](https://docs.mongodb.com/manual/reference/operator/query/nearSphere/#mongodb-query-op.-nearSphere) | Returns geospatial objects in proximity to a point on a sphere. Requires a geospatial index. The [2dsphere](https://docs.mongodb.com/manual/core/2dsphere/) and [2d](https://docs.mongodb.com/manual/core/2d/) indexes support [`$nearSphere`](https://docs.mongodb.com/manual/reference/operator/query/nearSphere/#mongodb-query-op.-nearSphere). |



### Array

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$all`](https://docs.mongodb.com/manual/reference/operator/query/all/#mongodb-query-op.-all) | Matches arrays that contain all elements specified in the query. |
| [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/#mongodb-query-op.-elemMatch) | Selects documents if element in the array field matches all the specified [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/#mongodb-query-op.-elemMatch) conditions. |
| [`$size`](https://docs.mongodb.com/manual/reference/operator/query/size/#mongodb-query-op.-size) | Selects documents if the array field is a specified size.    |

### Bitwise

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$bitsAllClear`](https://docs.mongodb.com/manual/reference/operator/query/bitsAllClear/#mongodb-query-op.-bitsAllClear) | Matches numeric or binary values in which a set of bit positions *all* have a value of `0`. |
| [`$bitsAllSet`](https://docs.mongodb.com/manual/reference/operator/query/bitsAllSet/#mongodb-query-op.-bitsAllSet) | Matches numeric or binary values in which a set of bit positions *all* have a value of `1`. |
| [`$bitsAnyClear`](https://docs.mongodb.com/manual/reference/operator/query/bitsAnyClear/#mongodb-query-op.-bitsAnyClear) | Matches numeric or binary values in which *any* bit from a set of bit positions has a value of `0`. |
| [`$bitsAnySet`](https://docs.mongodb.com/manual/reference/operator/query/bitsAnySet/#mongodb-query-op.-bitsAnySet) | Matches numeric or binary values in which *any* bit from a set of bit positions has a value of `1`. |



## Projection Operators

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$`](https://docs.mongodb.com/manual/reference/operator/projection/positional/#mongodb-projection-proj.-) | Projects the first element in an array that matches the query condition. |
| [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/projection/elemMatch/#mongodb-projection-proj.-elemMatch) | Projects the first element in an array that matches the specified [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/projection/elemMatch/#mongodb-projection-proj.-elemMatch) condition. |
| [`$meta`](https://docs.mongodb.com/manual/reference/operator/aggregation/meta/#mongodb-expression-exp.-meta) | Projects the document's score assigned during [`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#mongodb-query-op.-text) operation. |
| [`$slice`](https://docs.mongodb.com/manual/reference/operator/projection/slice/#mongodb-projection-proj.-slice) | Limits the number of elements projected from an array. Supports skip and limit slices. |



## Miscellaneous Operators

| Name                                                         | Description                               |
| :----------------------------------------------------------- | :---------------------------------------- |
| [`$comment`](https://docs.mongodb.com/manual/reference/operator/query/comment/#mongodb-query-op.-comment) | Adds a comment to a query predicate.      |
| [`$rand`](https://docs.mongodb.com/manual/reference/operator/query/rand/#mongodb-query-op.-rand) | Generates a random float between 0 and 1. |

