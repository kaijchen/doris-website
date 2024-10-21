---
{
    "title": "Transparent Rewriting with Sync-Materialized View",
    "language": "en"
}
---

<!-- 
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

## Principle

A sync-materialized view is a special type of table that precomputes and stores data based on a predefined SELECT statement. Its primary purpose is to satisfy users' analytical needs for arbitrary dimensions of raw detailed data while also enabling rapid fixed-dimension analytical queries.

Materialized views are suitable for the following scenarios:

1. Analytical requirements cover both detailed data queries and fixed-dimension queries.

2. Queries only involve a small subset of columns or rows in the table.

3. Queries contain time-consuming processing operations, such as long aggregation operations.

4. Queries require matching different prefix indexes.

For queries that frequently reuse the same subquery results, a sync-materialized view can significantly enhance performance. Doris automatically maintains the data in the materialized view, ensuring data consistency between the base table and the materialized view without additional manual maintenance costs. During queries, the system automatically matches the optimal materialized view and reads data directly from it.

When using materialized views, please note the following points:

1. In Doris version 2.0, materialized views have enhanced features. It is recommended that users confirm in a test environment whether the expected queries can hit the desired materialized views before using them in a formal production environment.

2. It is not advisable to create multiple similar materialized views on the same table, as this may lead to conflicts between them, resulting in query misses.

## Usage Process

The usage process for materialized views is as follows:

### 1 Create a Materialized View

1. Determine the type of materialized view to create based on the characteristics of the query statement.

2. Extract the common grouping and aggregation methods from multiple queries as the basis for defining the materialized view.

3. It is not necessary to create materialized views for all dimension combinations; only create them for commonly used dimension combinations.

4. Creating a materialized view is an asynchronous operation. After submitting the creation task, Doris will compute the existing data in the background until the creation is successful.

### 2 Automatic Query Matching

1. After the materialized view is successfully created, when a user queries the base table, Doris will automatically select an optimal materialized view and read data from it for computation.

2. Users can use the EXPLAIN command to check whether the current query is using a materialized view.

### 3 Update Strategy

To ensure data consistency between the materialized view and the base table, Doris synchronizes operations on the base table to the materialized view, using incremental updates to improve update efficiency and ensuring the atomicity of operations through transactions.

### 4 Supported Aggregation Functions

1. SUM, MIN, MAX (applicable to Version 0.12)

2. COUNT, BITMAP_UNION, HLL_UNION (applicable to Version 0.13)

3. General aggregation functions (applicable to Version 2.0)

## Tuning Usage Case

The following is a specific example to illustrate the use of single-table materialized views:

Suppose we have a detailed sales record table `sales_records` that records various information for each transaction, including transaction ID, salesperson ID, store ID, sales date, and transaction amount. Now, we frequently need to perform analytical queries on sales volumes for different stores.

To optimize the performance of these queries, we can create a materialized view `store_amt` that groups by store ID and sums the sales amounts for the same store. The specific steps are as follows:

### Create a Materialized View

First, we use the following SQL statement to create the materialized view `store_amt`:

```sql
CREATE MATERIALIZED VIEW store_amt AS 
SELECT store_id, SUM(sale_amt) 
FROM sales_records
GROUP BY store_id;
```

After submitting the creation task, Doris will asynchronously build this materialized view in the background. We can view the creation progress of the materialized view through the following command:

```sql
SHOW ALTER TABLE MATERIALIZED VIEW FROM db_name; 
```

When the `State` field changes to `FINISHED`, it indicates that the `store_amt` materialized view has been successfully created.

### Query Data

After the materialized view is created, when we query the sales volumes of different stores, Doris will automatically match the `store_amt` materialized view and read the pre-aggregated data directly from it, significantly improving query efficiency.

The query statement is as follows:

```sql
SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
```

We can also use the `EXPLAIN` command to check whether the query successfully hits the materialized view:

```sql
EXPLAIN SELECT store_id, SUM(sale_amt) FROM sales_records GROUP BY store_id;
```

At the end of the execution plan, if similar content is displayed, it indicates that the query successfully hits the `store_amt` materialized view:

```sql
TABLE: default_cluster:test.sales_records(store_amt), PREAGGREGATION: ON
```

By following these steps, we can utilize single-table materialized views to optimize query performance and improve the efficiency of data analysis.

## Summary

By creating materialized views, we can significantly enhance the query speed for related aggregation analyses. Materialized views not only enable us to perform statistical analyses quickly but also flexibly support the query requirements of detailed data, making them a very powerful feature in Doris.