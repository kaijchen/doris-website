---
{
    "title": "Transaction",
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

A transaction is an operation that contains one or more SQL statements. The execution of these statements must either be completely successful or completely fail. It is an indivisible work unit.

## Introduction

Queries and DDL single statements are implicit transactions and are not supported within multi-statement transactions. Each individual write is an implicit transaction by default, and multiple writes can form an explicit transaction. Currently, Doris does not support nested transactions.

## Explicit and Implicit Transactions

### Explicit Transactions

Explicit transactions require users to actively start, commit transactions. Only insert into values statement is supported in 2.1.

    ```sql
    BEGIN;
    [INSERT INTO VALUES]
    COMMIT;
    ```
Rollback is not supported in 2.1.

### Implicit Transactions

Implicit transactions refer to SQL statements that are executed without explicitly adding statements to start and commit transactions before and after the statements.

In Doris, except for [Group Commit](import/import-way/group-commit-manual.md), each import statement opens a transaction when it starts executing. The transaction is automatically committed after the statement is executed, or automatically rolled back if the statement fails. Each query or DDL statement is also an implicit transaction.

### Isolation Level

The only isolation level currently supported by Doris is READ COMMITTED. Under the READ COMMITTED isolation level, a statement sees only data that was committed before the statement began execution. It does not see uncommitted data.

When a single statement is executed, it captures a snapshot of the tables involved at the start of the statement, meaning that a single statement can only see commits from other transactions made before it began execution. Other transactions' commits are not visible during the execution of a single statement.

When a statement is executed inside a multi-statement transaction:

* It sees only data that was committed before the statement began execution. If another transaction commits between the execution of the first and the second statements, two successive statements in the same transaction may see different data.
* Currently, it cannot see changes made by previous statements within the same transaction.

### No Duplicates, No Loss

Doris supports mechanisms to ensure no duplicates and no loss during data writes. The Label mechanism ensures no duplicates within a single transaction, while two-phase commit coordinates to prevent duplicates across multiple transactions.

#### Label Mechanism

Transactions or writes in Doris can be assigned a Label. This Label is typically a user-defined string with some business logic attributes. If not set, a UUID string will be generated internally. The main purpose of a Label is to uniquely identify a transaction or import task and ensure that a transaction or import with the same Label will only execute successfully once. The Label mechanism ensures that data imports are neither lost nor duplicated. If the upstream data source guarantees at-least-once semantics, combined with Doris's Label mechanism, exactly-once semantics can be achieved. Labels are unique within a database.

Doris will clean up Labels based on time and number. By default, if the number of Labels exceeds 2000, cleanup will be triggered. Labels older than three days will also be cleaned up by default. Once a Label is cleaned up, a Label with the same name can execute successfully again, meaning it no longer has deduplication semantics.

Labels are usually set in the format of `business_logic+timestamp`, such as `my_business1_20220330_125000`. This Label typically represents a batch of data generated by the business `my_business1` at `2022-03-30 12:50:00`. By setting Labels this way, the business can query the import task status using the Label to clearly determine whether the batch of data at that time has been successfully imported. If not, the import can be retried using the same Label.

#### StreamLoad 2PC

[StreamLoad 2PC](#stream-load) is mainly used to support exactly-once semantics (EOS) when writing to Doris with Flink.

## Transaction Operations

### Start a Transaction

```sql
BEGIN;

BEGIN WITH LABEL {user_label}; 
```

If this statement is executed while the current session is in the middle of a transaction, Doris will ignore the statement, which can also be understood as transactions cannot be nested.

### Commit a Transaction

```sql
COMMIT;
```

Used to commit all modifications made in the current transaction.

## Transaction with multiple sql statements

Currently, Doris supports only one way of transaction loading.

### Multiple `INSERT INTO VALUES` for one table

Suppose the table schema is:

```sql
CREATE TABLE `dt` (
    `id` INT(11) NOT NULL,
    `name` VARCHAR(50) NULL,
    `score` INT(11) NULL
) ENGINE=OLAP
UNIQUE KEY(`id`)
DISTRIBUTED BY HASH(`id`) BUCKETS 1
PROPERTIES (
    "replication_num" = "1"
);
```

Do transaction load:

```sql
mysql> BEGIN;
Query OK, 0 rows affected (0.01 sec)
{'label':'txn_insert_b55db21aad7451b-b5b6c339704920c5', 'status':'PREPARE', 'txnId':''}

mysql> INSERT INTO dt (id, name, score) VALUES (1, "Emily", 25), (2, "Benjamin", 35), (3, "Olivia", 28), (4, "Alexander", 60), (5, "Ava", 17);
Query OK, 5 rows affected (0.08 sec)
{'label':'txn_insert_b55db21aad7451b-b5b6c339704920c5', 'status':'PREPARE', 'txnId':'10013'}

mysql> INSERT INTO dt VALUES (6, "William", 69), (7, "Sophia", 32), (8, "James", 64), (9, "Emma", 37), (10, "Liam", 64);
Query OK, 5 rows affected (0.00 sec)
{'label':'txn_insert_b55db21aad7451b-b5b6c339704920c5', 'status':'PREPARE', 'txnId':'10013'}

mysql> COMMIT;
Query OK, 0 rows affected (1.02 sec)
{'label':'txn_insert_b55db21aad7451b-b5b6c339704920c5', 'status':'VISIBLE', 'txnId':'10013'}
```

This method not only achieves atomicity, but also in Doris, it enhances the writing performance of `INSERT INTO VALUES`.

If user enables `Group Commit` and transaction insert at the same time, the transaction insert will work. 

#### QA

* Writing to multiple tables must belong to the same Database; otherwise, you will encounter the error `Transaction insert must be in the same database`

* If the time-consuming from `BEGIN` statement exceeds the timeout configured in Doris, the transaction will be rolled back. Currently, the timeout uses the maximum value of session variables `insert_timeout` and `query_timeout`.

* When using JDBC to connect to Doris for transaction operations, please add `useLocalSessionState=true` in the JDBC URL; otherwise, you may encounter the error `This is in a transaction, only insert, update, delete, commit, rollback is acceptable`.

## Stream Load 2PC

**1. Enable two-phase commit by setting `two_phase_commit:true` in the HTTP Header.**

```shell
curl --location-trusted -u user:passwd -H "two_phase_commit:true" -T test.txt http://fe_host:http_port/api/{db}/{table}/_stream_load
{
    "TxnId": 18036,
    "Label": "55c8ffc9-1c40-4d51-b75e-f2265b3602ef",
    "TwoPhaseCommit": "true",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 100,
    "NumberLoadedRows": 100,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 1031,
    "LoadTimeMs": 77,
    "BeginTxnTimeMs": 1,
    "StreamLoadPutTimeMs": 1,
    "ReadDataTimeMs": 0,
    "WriteDataTimeMs": 58,
    "CommitAndPublishTimeMs": 0
}
```

**2. Trigger the commit operation for a transaction (can be sent to FE or BE).**

- Specify the transaction using the Transaction ID:

  ```shell
  curl -X PUT --location-trusted -u user:passwd -H "txn_id:18036" -H "txn_operation:commit" http://fe_host:http_port/api/{db}/{table}/stream_load2pc
  {
      "status": "Success",
      "msg": "transaction [18036] commit successfully."
  }
  ```

- Specify the transaction using the label:

  ```shell
  curl -X PUT --location-trusted -u user:passwd -H "label:55c8ffc9-1c40-4d51-b75e-f2265b3602ef" -H "txn_operation:commit"  http://fe_host:http_port/api/{db}/{table}/_stream_load_2pc
  {
      "status": "Success",
      "msg": "label [55c8ffc9-1c40-4d51-b75e-f2265b3602ef] commit successfully."
  }
  ```

**3. Trigger the abort operation for a transaction (can be sent to FE or BE).**

- Specify the transaction using the Transaction ID:

  ```shell
  curl -X PUT --location-trusted -u user:passwd -H "txn_id:18037" -H "txn_operation:abort"  http://fe_host:http_port/api/{db}/{table}/stream_load2pc
  {
      "status": "Success",
      "msg": "transaction [18037] abort successfully."
  }
  ```

- Specify the transaction using the label:

  ```shell
  curl -X PUT --location-trusted -u user:passwd -H "label:55c8ffc9-1c40-4d51-b75e-f2265b3602ef" -H "txn_operation:abort"  http://fe_host:http_port/api/{db}/{table}/stream_load2pc
  {
      "status": "Success",
      "msg": "label [55c8ffc9-1c40-4d51-b75e-f2265b3602ef] abort successfully."
  }
  ```

## Broker Load into muti tables with a transaction

All Broker Load tasks are atomic and ensure atomicity even when loading multiple tables within the same task. The Label mechanism can be used to ensure data load without loss or duplication.

The following example demonstrates loading data from HDFS by using wildcard patterns to match two sets of files and load them into two different tables.

```sql
LOAD LABEL example_db.label2
(
    DATA INFILE("hdfs://hdfs_host:hdfs_port/input/file-10*")
    INTO TABLE `my_table1`
    PARTITION (p1)
    COLUMNS TERMINATED BY ","
    (k1, tmp_k2, tmp_k3)
    SET (
        k2 = tmp_k2 + 1,
        k3 = tmp_k3 + 1
    )
    DATA INFILE("hdfs://hdfs_host:hdfs_port/input/file-20*")
    INTO TABLE `my_table2`
    COLUMNS TERMINATED BY ","
    (k1, k2, k3)
)
WITH BROKER hdfs
(
    "username"="hdfs_user",
    "password"="hdfs_password"
);
```

The wildcard pattern is used to match and load two sets of files, `file-10*` and `file-20*`, into `my_table1` and `my_table2` respectively. In the case of `my_table1`, the load is specified to the `p1` partition, and the values of thesecond and third columns in the source file are incremented by 1 before being loaded.