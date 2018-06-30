

# 事务 Transactions

[TOC]

## 译者前言：

数据模型定义对照：

| MySQL  | MongoDB |
| ------ | ------- |
| 行     | 文档    |
| 表     | 集合    |
| 库     | 库      |
| 组复制 | 复制集  |

专有名词翻译约定：

| 中文     | 原文         |
| -------- | ------------ |
| 读一致性 | readConcern  |
| 写一致性 | writeConcern |

## 简介：

4.0版本中加入

New in version 4.0

在MongoDB中，针对单个文档的操作是原子性的。由于MongoDB允许在单个文档中嵌入用以表示相互之间关系的子文档和数组来替代跨文档和集合的连接操作，这种折中的方式在很多场景下间接实现了多文档事务的特性。

In MongoDB, an operation on a single document is atomic. Because you can use embedded documents and arrays to capture relationships between data in a single document structure instead of normalizing across multiple documents and collections, this single-document atomicity obviates the need for multi-document transactions for many practical use cases.

但这种办法在面对多文档同时更新或者多文档一致性读的时候就显得捉襟见肘，MongoDB新版本提供了面向复制集的多文档事务特性。其能满足在多个操作，文档，集合，数据库之间的事务性，事务的特性：一个事务中的若干个操作要么全部完成，要么全部回滚，操作的原子性A，数据更新的一致性C。事务提交时，所有数据更改都会被永久保存D。事务提交前其数据更改不会被外部获取到I。

However, for situations that require atomicity for updates to multiple documents or consistency between reads to multiple documents, MongoDB provides the ability to perform multi-document transactions against replica sets. Multi-document transactions can be used across multiple operations, collections, databases, and documents. Multi-document transactions provide an “all-or-nothing” proposition. When a transaction commits, all data changes made in the transaction are saved. If any operation in the transaction fails, the transaction aborts and all data changes made in the transaction are discarded without ever becoming visible. Until a transaction commits, no write operations in the transaction are visible outside the transaction.

注意：

在多数场景下，多文档事务相对于单文档数据变更性能损耗会更严重。且不要因为支持多文档事务就妄图放松对数据结构设计的要求。无论哪种数据库，巧妙的表设计总是会比头铁强行使用不必要的低级数据关联来的高效。

一般情况下，由于减少了对多文档事务的需求，非范式化(即内嵌子文档和数组)的数据模型还是会发挥比较大的作用

In most cases, multi-document transaction incurs a greater performance cost over single document writes, and the availability of multi-document transaction should not be a replacement for effective schema design. For many scenarios, the [denormalized data model (embedded documents and arrays)](https://docs.mongodb.com/manual/core/data-model-design/#data-modeling-embedding) will continue to be optimal for your data and use cases. That is, for many scenarios, modeling your data appropriately will minimize the need for multi-document transactions.

## 事务和复制集 Transactions and Replica Sets

多文档事务在4.0版本仅支持复制集，对分片集群的事务性支持计划在4.2版本中实现。

Multi-document transactions are available for replica sets only. Transactions for sharded clusters are scheduled for MongoDB 4.2 [[1\]](https://docs.mongodb.com/manual/core/transactions/#upcoming).

| 免责声明，不翻译了 | The development, release, and timing of any features or functionality described for our products remains at our sole discretion. This information is merely intended to outline our general product direction and it should not be relied on in making a purchasing decision nor is this a commitment, promise or legal obligation to deliver any material, code, or functionality. |
| ------------------ | ------------------------------------------------------------ |
|                    |                                                              |

### 新特性版本向后兼容度 Feature Compatibility Version (FCV)

想使用多文档事务的特性的话， `featureCompatibilityVersion`值必须设为`4.0`以上。(开启后，既存的数据不支持用4.0以下版本的mongod启动)

The `featureCompatibilityVersion` (fCV) of all members of the replica set must be `4.0` or greater. To check the fCV for a member, connect to the member and run the following command:

```mysql
db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
```

For more information on fCV, see [`setFeatureCompatibilityVersion`](https://docs.mongodb.com/manual/reference/command/setFeatureCompatibilityVersion/#dbcmd.setFeatureCompatibilityVersion).

### 存储引擎 Storage Engines

多文档事务特性仅支持WiredTiger存储引擎

Multi-document transactions are only available for deployments that use WiredTiger storage engine.

Multi-document transactions are not available for deployments that use in-memory storage engine or the deprecated MMAPv1 storage engine.

## 事务与文档操作 Transactions and Operations

注意事项：

For transactions:

​	可以在任何跨库的表上进行CURD操作

- You can specify read/write (CRUD) operations on **existing** collections. The collections can be in different databases.

  `config`,`admin`,`local`库中的表不支持多文档事务

- You cannot read/write to collections in the `config`, `admin`, or `local` databases.

  各库中的`system.*`表不支持多文档事务

- You cannot write to `system.*` collections.

  在当前会话的事务中无法进行返回当前操作的查询计划

- You cannot return the supported operation’s query plan (i.e. `explain`).

  在事务外创建的游标无法在事务中进行`getMore`操作

- For cursors created outside of transactions, you cannot call [`getMore`](https://docs.mongodb.com/manual/reference/command/getMore/#dbcmd.getMore) inside a transaction.

  在事务中创建的游标无法在事务外进行`getMore`操作

- For cursors created in a transaction, you cannot call [`getMore`](https://docs.mongodb.com/manual/reference/command/getMore/#dbcmd.getMore) outside the transaction.

在多文档事务中无法进行诸如创建或者删除集合，添加索引等更新数据库元数据的操作。进一步的说，在多文档事务中那些可能会隐式创建集合的操作也是被禁止的。

Operations that affect the database catalog, such as creating or dropping a collection or an index, are not allowed in multi-document transactions. For example, a multi-document transaction cannot include an insert operation that would result in the creation of a new collection. See [Restricted Operations](https://docs.mongodb.com/manual/core/transactions/#transactions-ops-restricted).

提示：

如果在创建或者删除集合后会紧接着开启一个会对该集合进行操作的事务，那么创建或者删除集合的操作需要追加写一致性（writeConcern）扩散到多数节点的参数以确保事务可以成功获取该集合相关的锁。

（注，这段话可能较为拗口。其实是从集群的一致性来考虑，和事务的读一致性readConcern相关）

When creating or dropping a collection immediately before starting a transaction, if the collection is accessed within the transaction, issue the create or drop operation with write concern [`"majority"`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%22majority%22) to ensure that the transaction can acquire the required locks.

目前支持多文档事务的命令和方法

For multi-document transactions:

| Method                                                       | Command                                                      | Note                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`db.collection.aggregate()`](https://docs.mongodb.com/manual/reference/method/db.collection.aggregate/#db.collection.aggregate) | [`aggregate`](https://docs.mongodb.com/manual/reference/command/aggregate/#dbcmd.aggregate) | Excluding the following stages:[`$collStats`](https://docs.mongodb.com/manual/reference/operator/aggregation/collStats/#pipe._S_collStats)[`$currentOp`](https://docs.mongodb.com/manual/reference/operator/aggregation/currentOp/#pipe._S_currentOp)[`$indexStats`](https://docs.mongodb.com/manual/reference/operator/aggregation/indexStats/#pipe._S_indexStats)[`$listLocalSessions`](https://docs.mongodb.com/manual/reference/operator/aggregation/listLocalSessions/#pipe._S_listLocalSessions)[`$listSessions`](https://docs.mongodb.com/manual/reference/operator/aggregation/listSessions/#pipe._S_listSessions)[`$out`](https://docs.mongodb.com/manual/reference/operator/aggregation/out/#pipe._S_out) |
| [`db.collection.distinct()`](https://docs.mongodb.com/manual/reference/method/db.collection.distinct/#db.collection.distinct) | [`distinct`](https://docs.mongodb.com/manual/reference/command/distinct/#dbcmd.distinct) |                                                              |
| [`db.collection.find()`](https://docs.mongodb.com/manual/reference/method/db.collection.find/#db.collection.find) | [`find`](https://docs.mongodb.com/manual/reference/command/find/#dbcmd.find) |                                                              |
|                                                              | [`geoSearch`](https://docs.mongodb.com/manual/reference/command/geoSearch/#dbcmd.geoSearch) |                                                              |
| [`db.collection.deleteMany()`](https://docs.mongodb.com/manual/reference/method/db.collection.deleteMany/#db.collection.deleteMany)[`db.collection.deleteOne()`](https://docs.mongodb.com/manual/reference/method/db.collection.deleteOne/#db.collection.deleteOne)[`db.collection.remove()`](https://docs.mongodb.com/manual/reference/method/db.collection.remove/#db.collection.remove) | [`delete`](https://docs.mongodb.com/manual/reference/command/delete/#dbcmd.delete) |                                                              |
| [`db.collection.findOneAndDelete()`](https://docs.mongodb.com/manual/reference/method/db.collection.findOneAndDelete/#db.collection.findOneAndDelete)[`db.collection.findOneAndReplace()`](https://docs.mongodb.com/manual/reference/method/db.collection.findOneAndReplace/#db.collection.findOneAndReplace)[`db.collection.findOneAndUpdate()`](https://docs.mongodb.com/manual/reference/method/db.collection.findOneAndUpdate/#db.collection.findOneAndUpdate) | [`findAndModify`](https://docs.mongodb.com/manual/reference/command/findAndModify/#dbcmd.findAndModify) | For `upsert`, only when run against an existing collection.  |
| [`db.collection.insertMany()`](https://docs.mongodb.com/manual/reference/method/db.collection.insertMany/#db.collection.insertMany)[`db.collection.insertOne()`](https://docs.mongodb.com/manual/reference/method/db.collection.insertOne/#db.collection.insertOne)[`db.collection.insert()`](https://docs.mongodb.com/manual/reference/method/db.collection.insert/#db.collection.insert) | [`insert`](https://docs.mongodb.com/manual/reference/command/insert/#dbcmd.insert) | Only when run against an existing collection.                |
| [`db.collection.save()`](https://docs.mongodb.com/manual/reference/method/db.collection.save/#db.collection.save) |                                                              | If an insert, only when run against an existing collection.  |
| [`db.collection.updateOne()`](https://docs.mongodb.com/manual/reference/method/db.collection.updateOne/#db.collection.updateOne)[`db.collection.updateMany()`](https://docs.mongodb.com/manual/reference/method/db.collection.updateMany/#db.collection.updateMany)[`db.collection.replaceOne()`](https://docs.mongodb.com/manual/reference/method/db.collection.replaceOne/#db.collection.replaceOne)[`db.collection.update()`](https://docs.mongodb.com/manual/reference/method/db.collection.update/#db.collection.update) | [`update`](https://docs.mongodb.com/manual/reference/command/update/#dbcmd.update) | For `upsert`, only when run against an existing collection.  |
| [`db.collection.bulkWrite()`](https://docs.mongodb.com/manual/reference/method/db.collection.bulkWrite/#db.collection.bulkWrite)Various [Bulk Operation Methods](https://docs.mongodb.com/manual/reference/method/js-bulk/) |                                                              | For insert operations, only when run against an existing collection. |

### 计数操作 Count Operation

如果需要在事务中进行计数操作，需要使用 [`$count`](https://docs.mongodb.com/manual/reference/operator/aggregation/count/#pipe._S_count) 操作符或者把 [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group) 和[`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum)结合起来

To perform a count operation within a transaction, use the [`$count`](https://docs.mongodb.com/manual/reference/operator/aggregation/count/#pipe._S_count) aggregation stage or the [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group) (with a[`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum) expression) aggregation stage.

兼容4.0版本的MongoDB驱动内含一个集合层级由 [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group) 和[`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum) 包装出来的 `countDocuments(filter,options)` 计数接口

MongoDB drivers compatible with the 4.0 features provide a collection-level API `countDocuments(filter,options)` as a helper method that uses the [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group) with a [`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum) expression to perform a count.



### 获取参数与信息相关的操作  Informational Operations

诸如 [`isMaster`](https://docs.mongodb.com/manual/reference/command/isMaster/#dbcmd.isMaster), [`buildInfo`](https://docs.mongodb.com/manual/reference/command/buildInfo/#dbcmd.buildInfo), [`connectionStatus`](https://docs.mongodb.com/manual/reference/command/connectionStatus/#dbcmd.connectionStatus) （及其相关的帮助命令）这些获取参数或者信息的命令可以在事务中使用，但不能作为事务中起手的第一个操作。

Informational commands, such as [`isMaster`](https://docs.mongodb.com/manual/reference/command/isMaster/#dbcmd.isMaster), [`buildInfo`](https://docs.mongodb.com/manual/reference/command/buildInfo/#dbcmd.buildInfo), [`connectionStatus`](https://docs.mongodb.com/manual/reference/command/connectionStatus/#dbcmd.connectionStatus) (and their helper methods) are allowed in transactions; however, they cannot be the first operation in the transaction.

### 受到限制的操作 Restricted Operations

下列操作不允许在多文档事务中使用：

The following operations are not allowed in multi-document transactions:

​	可能会影响到数据库元信息的操作，诸如创建或者删除集合，增加索引。（注：其实就是DDL操作吧）。

- Operations that affect the database catalog, such as creating or dropping a collection or an index. For example, a multi-document transaction cannot include an insert operation that would result in the creation of a new collection.

   [`listCollections`](https://docs.mongodb.com/manual/reference/command/listCollections/#dbcmd.listCollections) 和 [`listIndexes`](https://docs.mongodb.com/manual/reference/command/listIndexes/#dbcmd.listIndexes) 及其相关的帮助命令都在上述禁止之列。（注：这个有点意外）

  The [`listCollections`](https://docs.mongodb.com/manual/reference/command/listCollections/#dbcmd.listCollections) and [`listIndexes`](https://docs.mongodb.com/manual/reference/command/listIndexes/#dbcmd.listIndexes) commands and their helper methods are also excluded.

  非增删改查和非查询参数与信息的操作，诸如 [`createUser`](https://docs.mongodb.com/manual/reference/command/createUser/#dbcmd.createUser), [`getParameter`](https://docs.mongodb.com/manual/reference/command/getParameter/#dbcmd.getParameter), [`count`](https://docs.mongodb.com/manual/reference/command/count/#dbcmd.count)及其帮助命令等等。

- Non-CRUD and non-informational operations, such as [`createUser`](https://docs.mongodb.com/manual/reference/command/createUser/#dbcmd.createUser), [`getParameter`](https://docs.mongodb.com/manual/reference/command/getParameter/#dbcmd.getParameter), [`count`](https://docs.mongodb.com/manual/reference/command/count/#dbcmd.count), etc. and their helpers.

## 事务与数据库安全相关 Transactions and Security

​	如果启用了鉴权，欲使用多文档事务，必须拥有操作事务的权限。

- If running with [access control](https://docs.mongodb.com/manual/core/authorization/), you must have privileges for the [operations in the transaction](https://docs.mongodb.com/manual/core/transactions/#transactions-operations). [[2\]](https://docs.mongodb.com/manual/core/transactions/#username-external)

  如果开启了审计，回滚的事务仍然会被记录。

- If running with [auditing](https://docs.mongodb.com/manual/core/auditing/), operations in an aborted transaction are still audited.

| 若使用了附加验证方式，用户名大小不能超过10K | If using `$external` authentication users (i.e. Kerberos, LDAP, x.509 users), the usernames cannot be greater than 10k bytes. |
| ------------------------------------------- | ------------------------------------------------------------ |
|                                             |                                                              |

## 事务与会话相关 Transactions and Sessions

事务是和会话紧密相关的，一个会话在其生命周期内最多只能并发一个处于激活状态的事务。

Transations are associated with a session. That is, you start a transaction for a session. At any given time, you can have at most one open transaction for a session.

重点 IMPORTANT

当使用驱动进行连接时（非mongo shell连接），必须将会话与每个事务对应起来。（注：多事务就要开启多个连接会话类）。若在事务正在进行时会话退出，则事务会进行回滚

When using the drivers, you **must** pass the session to each operation in the transaction.

If a session ends and it has an open transaction, the transaction aborts.

## 事务与驱动的兼容 Transactions and MongoDB Drivers

以下是支持多文档事务的各语言最低驱动版本号：

Clients require MongoDB drivers updated for MongoDB 4.0.

| Java  |    Python     |  C#  | Node  | Ruby  | Perl  | PHPC  | Scala |
| :---: | :-----------: | :--: | :---: | :---: | ----- | ----- | ----- |
| 3.8.0 | 3.7.0C 1.11.0 | 2.7  | 3.1.0 | 2.6.0 | 2.0.0 | 1.5.0 | 2.4.0 |

重点：IMPORTANT

如果想在事务中进行读写结合的操作，同样必须把每个操作和会话对应起来

To associate read and write operations with a transaction, you **must** pass the session to each operation in the transaction. For examples, see [Transactions and Retryable Writes](https://docs.mongodb.com/manual/core/transactions/#transactions-retry).

## 事务与mongo shell工具 Transactions and the `mongo` Shell

下面这些mongo shell方法可以用于操作事务：

The following [`mongo`](https://docs.mongodb.com/manual/reference/program/mongo/#bin.mongo) shell methods are available for transactions:

- [`Session.startTransaction()`](https://docs.mongodb.com/manual/reference/method/Session.startTransaction/#Session.startTransaction)
- [`Session.commitTransaction()`](https://docs.mongodb.com/manual/reference/method/Session.commitTransaction/#Session.commitTransaction)
- [`Session.abortTransaction()`](https://docs.mongodb.com/manual/reference/method/Session.abortTransaction/#Session.abortTransaction)

## 事务和重写 Transactions and Retryable Writes

面向高可用应用 HIGHLY AVAILABLE APPLICATIONS

无论是MongoDB还是关系型数据库，与其连接的应用都应该设法处理事务提交过程的错误，设计事务的重新提做逻辑。

Regardless of the database system, whether MongoDB or relational databases, applications should take measures to handle errors during transaction commits and incorporate retry logic for transactions.

### 事务重做 Retry Transaction

无论 [`retryWrites`](https://docs.mongodb.com/manual/reference/connection-string/#urioption.retryWrites) 有没有被设置为`true`，事务中独立的写操作总是是无法被重做的

The individual write operations inside the transaction are not retryable, regardless of whether [`retryWrites`](https://docs.mongodb.com/manual/reference/connection-string/#urioption.retryWrites) is set to `true`.

如果操作执行时发生了错误，会返回一个包含名为 `errorLabels` 的数组字段。如果该错误是暂时性的， `errorLabels` 会包含一个 `"TransientTransactionError"`元素，标志着整个事务可以被重做。

If an operation encounters an error, the returned error may have an `errorLabels` array field. If the error is a transient error, the `errorLabels` array field contains `"TransientTransactionError"` as an element and the transaction as a whole can be retried.

例如，下面的下面的JAVA函数就实现了执行事务并重做那些包含`"TransientTransactionError"`的错误。

（注：官方文档有其他几种语言的例程，这里选择使用人数最多的JAVA）

For example, the following helper runs a function and retries the function if a `"TransientTransactionError"` is encountered:

```java
void runTransactionWithRetry(Runnable transactional) {
    while (true) {
        try {
            transactional.run();
            break;
        } catch (MongoException e) {
            System.out.println("Transaction aborted. Caught exception during transaction.");

            if (e.hasErrorLabel(MongoException.TRANSIENT_TRANSACTION_ERROR_LABEL)) {
                System.out.println("TransientTransactionError, aborting transaction and retrying ...");
                continue;
            } else {
                throw e;
            }
        }
    }
}
```

### 提交操作可以被重做 Retry Commit Operation

提交操作本身是可以被重做的。如果事务在提交过程中遇到错误，驱动会自动忽略 `retryWrites`的设置进行一次重试。

The commit operations are [retryable write operations](https://docs.mongodb.com/manual/core/retryable-writes/). If the commit operation operation encounters an error, MongoDB drivers retry the operation a single time regardless of whether [`retryWrites`](https://docs.mongodb.com/manual/reference/connection-string/#urioption.retryWrites) is set to `true`.

如果事务提交的过程中发生错误，MongoDB会返回一个包含 `errorLabels` 的数组字段。如果错误是暂时性的， `errorLabels` 字段会包含一个`"UnknownTransactionCommitResult"`元素，标识这个事务可以被重新提交

If the commit operation encounters an error, MongoDB returns an error with an `errorLabels` array field. If the error is a transient commit error, the `errorLabels` array field contains`"UnknownTransactionCommitResult"` as an element and the commit operation can be retried.

尽管MongoDB驱动提供了一次事务重新提交机制，应用方面仍应该设计一个方法去处理事务提交过程中爆出的错误。

In addition to the single retry behavior provided by the MongoDB drivers, applications should take measures to handle `"UnknownTransactionCommitResult"` errors during transaction commits.

仍以JAVA为例：

```java
void commitWithRetry(ClientSession clientSession) {
    while (true) {
        try {
            clientSession.commitTransaction();
            System.out.println("Transaction committed");
            break;
        } catch (MongoException e) {
            // can retry commit
            if (e.hasErrorLabel(MongoException.UNKNOWN_TRANSACTION_COMMIT_RESULT_LABEL)) {
                System.out.println("UnknownTransactionCommitResult, retrying commit operation ...");
                continue;
            } else {
                System.out.println("Exception during commit ...");
                throw e;
            }
        }
    }
}
```

### 事务与提交操作的重做 Retry Transaction and Commit Operation

将上文两个逻辑结合起来，看下面的例程：（JAVA）

```JAVa
void runTransactionWithRetry(Runnable transactional) {
    while (true) {
        try {
            transactional.run();
            break;
        } catch (MongoException e) {
            System.out.println("Transaction aborted. Caught exception during transaction.");

            if (e.hasErrorLabel(MongoException.TRANSIENT_TRANSACTION_ERROR_LABEL)) {
                System.out.println("TransientTransactionError, aborting transaction and retrying ...");
                continue;
            } else {
                throw e;
            }
        }
    }
}

void commitWithRetry(ClientSession clientSession) {
    while (true) {
        try {
            clientSession.commitTransaction();
            System.out.println("Transaction committed");
            break;
        } catch (MongoException e) {
            // can retry commit
            if (e.hasErrorLabel(MongoException.UNKNOWN_TRANSACTION_COMMIT_RESULT_LABEL)) {
                System.out.println("UnknownTransactionCommitResult, retrying commit operation ...");
                continue;
            } else {
                System.out.println("Exception during commit ...");
                throw e;
            }
        }
    }
}

void updateEmployeeInfo() {
    MongoCollection<Document> employeesCollection = client.getDatabase("hr").getCollection("employees");
    MongoCollection<Document> eventsCollection = client.getDatabase("hr").getCollection("events");

    try (ClientSession clientSession = client.startSession()) {
        clientSession.startTransaction();

        employeesCollection.updateOne(clientSession,
                Filters.eq("employee", 3),
                Updates.set("status", "Inactive"));
        eventsCollection.insertOne(clientSession,
                new Document("employee", 3).append("status", new Document("new", "Inactive").append("old", "Active")));

        commitWithRetry(clientSession);
    }
}


void updateEmployeeInfoWithRetry() {
    runTransactionWithRetry(this::updateEmployeeInfo);
}
```

## 原子性 Atomicity

多文档事务满足ACID的原子性

Multi-document transactions are atomic:

​	当一个事务被提交时，该事务内部中所有的变更都会保存并且可以被其他会话的事务读到。

- When a transaction commits, all data changes made in the transaction are saved and visible outside the transaction. Until a transaction commits, the data changes made in the transaction are not visible outside the transaction.

  当事务回滚时，事务内部所有的变更都会被干净的丢弃。比如，事务中的任一操作失败后，事务回滚，当前事务中所有的变更都会被丢弃，且不为其他会话所知。

- When a transaction aborts, all data changes made in the transaction are discarded without ever becoming visible. For example, if any operation in the transaction fails, the transaction aborts and all data changes made in the transaction are discarded without ever becoming visible.

## 读倾向性 Read Preference

多文档事务中的读操作必须使用`primary`倾向性，即从复制集的主实例

[Multi-document transactions](https://docs.mongodb.com/manual/core/transactions/#) that contain read operations must use read preference [`primary`](https://docs.mongodb.com/manual/reference/read-preference/#primary).

All operations in a given transaction must route to the same member.

## 事务参数选项 Transaction Options (Read Concern/Write Concern)

```mysql
session.startTransaction( {
   readConcern: { level: <level> },
   writeConcern: { w: <value>, j: <boolean>, wtimeout: <number> }
} );
```

### 读一致性 Read Concern

多文档事务支持三种读一致性等级： [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22), [`"local"`](https://docs.mongodb.com/manual/reference/read-concern-local/#readconcern.%22local%22), and [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22)，即快照，本机，多数派

Multi-document transactions support read concern [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22), [`"local"`](https://docs.mongodb.com/manual/reference/read-concern-local/#readconcern.%22local%22), and [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22):

​	针对本机和多数派的读一致性等级，MongoDB在有些时候回使用更强的等级进行替换。（注：为啥？）

- For [`"local"`](https://docs.mongodb.com/manual/reference/read-concern-local/#readconcern.%22local%22) and [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22) read concern, MongoDB may sometimes substitute a stronger read concern.

  针对多数派读一致性等级，如果事务使用此级别进行提交，事务操作都会读到副本集中多数派已经提交的数据。

- For [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22) read concern, if the transaction commits with [write concern “majority”](https://docs.mongodb.com/manual/core/transactions/#transactions-write-concern), transaction operations are guaranteed to have read majority-committed data. Otherwise, the [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22) read concern provides no guarantees that read operations read majority-committed data.

  针对快照级别的读一致性，如果。。。。。【官方文档这儿是不是写错了】，如果事务使用快照级别的读一致性进行提交，那么可以保证事务中的操作读到的都是事务开始时的多数派事务快照。

- For [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22) read concern, if the transaction commits with [write concern “majority”](https://docs.mongodb.com/manual/core/transactions/#transactions-write-concern), the transaction operations are guaranteed to have read from a snapshot of majority committed data. Otherwise, the [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22) read concern provides no guarantee that read operations used a snapshot of majority-committed data.

如果针对当前事务设置读一致性，而不是对事务中的独立操作设置一致性，那该一致性会覆盖事务中所有的事务。而且在事务中，事务粒度的一致性会覆盖集合级别，数据库级别，客户端级别的读一致性。

You set the read concern at the transaction level, not at the individual operation level. The operations in the transaction will use the transaction-level read concern. Any read concern set at the collection and database level is ignored inside the transaction. If the transaction-level read concern is explicitly specified, the client level read concern is also ignored inside the transaction.

可以在事务开始时对读一致性进行设置。

You can set the transaction [read concern](https://docs.mongodb.com/manual/reference/read-concern/) at the transaction start.

如果缺省了事务级别的读一致性设置，事务会继承会话级别的读一致性，如果会话级别也没有设置读一致性，那么会继续向上继承客y户端层级设置的读一致性。

If unspecified at the transaction start, transactions use the session-level read concern or, if that is unset, the client-level read concern.

### 写一致性 Write Concern

如果针对当前事务设置写一致性，而不是对事务中的独立操作设置一致性。那么在提交时，事务会使用自身的一致性去提交写操作。事务中各个独立操作忽略写一致性，请不要徒劳设置。（注：这段原文语义模糊，这里认为本段原文与上段原文写法不同，是为了和上段中的情况进行区分）

You set the write concern at the transaction level, not at the individual operation level. At the time of the commit, transactions use the transaction level [write concern](https://docs.mongodb.com/manual/reference/write-concern/) to commit the write operations. Individual operations inside the transaction ignore write concerns. Do not explicitly set the write concern for the individual write operations inside transactions.

可以在事务开始时对读一致性进行设置。

You can set the [write concern](https://docs.mongodb.com/manual/reference/write-concern/) for the transaction commit at the transaction start.

​	如果缺省了事务级别的写一致性设置，那么事务在提交时会继承会话级别的写一致性，会话级别也缺省了设置的话，会继续向上继承客户端级别的写一致性。

- If unspecified at the transaction start, transactions use the session-level write concern for the commit or, if that is unset, the client-level write concern.

  不支持写一致性设置为[`w: 0`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%3Cnumber%3E) 

- Write concern [`w: 0`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%3Cnumber%3E) is not supported for transactions.

  如果使用 [`w: 1`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%3Cnumber%3E) 进行提交，那么该事务提交时只会确认本机oK,也就是PrimaryOK。如果主节点挂掉，且该事务没有被传输到当时的次级节点。那么，恭喜你，该事务大概率会被回滚掉。会造成数据库端和应用端数据的不一致和数据缺失。

- If you commit using [`w: 1`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%3Cnumber%3E) write concern, your transaction can be [rolled back if there is a failover](https://docs.mongodb.com/manual/core/replica-set-rollbacks/).

  如果事务使用多数派写一致性进行提交，且指定了快照级别的读一致性。那么事务操作会确保从事务开始时多数派已经提交的快照数据中进行读取。（注：这段语义不明，官方文档到底在表达什么，是不是写错了）

- If the transaction commits with [write concern “majority”](https://docs.mongodb.com/manual/core/transactions/#transactions-write-concern) and has specified read concern [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22) read concern, transaction operations are guaranteed to have read from a snapshot of majority-committed data. Otherwise, the [`"snapshot"`](https://docs.mongodb.com/manual/reference/read-concern-snapshot/#readconcern.%22snapshot%22) read concern provides no guarantees that read operations used a snapshot of majority-committed data.

  如果事务使用多数派写一致性进行提交，且指定了多数派级别的读一致性，那么事务中的操作会被确保从多数派已提交的数据中进行读取。（注：这个是不是官方文档编辑时复制粘贴错了，把上面读一致性的内容放过来了）

- If the transaction commits with [write concern “majority”](https://docs.mongodb.com/manual/core/transactions/#transactions-write-concern) and has specified read concern [`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22) read concern, transaction operations are guaranteed to have read majority-committed data. Otherwise, the[`"majority"`](https://docs.mongodb.com/manual/reference/read-concern-majority/#readconcern.%22majority%22) read concern provides no guarantees that read operations read majority-committed data.

## 事务和锁 Transactions and Locks

事务中的操作在获取其所需要数据上的锁时，默认等待超时时间为5毫秒，超时后，当前事务会被回滚。（注：这个太短了吧，幸好可以手动设置）

By default, transactions waits `5` milliseconds to acquire locks required by the operations in the transaction. If the transaction cannot acquire its required locks within the `5` milliseconds, the transaction aborts.

提示：

如果在创建或者删除集合后会紧接着开启一个会对该集合进行操作的事务，那么创建或者删除集合的操作需要追加写一致性（writeConcern）扩散到多数节点的参数以确保事务可以成功获取该集合相关的锁。

（注，这段话可能较为拗口。其实是从集群的一致性来考虑，和事务的读一致性readConcern相关）

When creating or dropping a collection immediately before starting a transaction, if the collection is accessed within the transaction, issue the create or drop operation with write concern [`"majority"`](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern.%22majority%22) to ensure that the transaction can acquire the required locks.

可以使用 [`maxTransactionLockRequestTimeoutMillis`](https://docs.mongodb.com/manual/reference/parameters/#param.maxTransactionLockRequestTimeoutMillis) 参数对锁等待超时长度进行设置。

You can use the [`maxTransactionLockRequestTimeoutMillis`](https://docs.mongodb.com/manual/reference/parameters/#param.maxTransactionLockRequestTimeoutMillis) parameter to adjust how long transactions wait to acquire locks.

也可以将 [`maxTransactionLockRequestTimeoutMillis`](https://docs.mongodb.com/manual/reference/parameters/#param.maxTransactionLockRequestTimeoutMillis) 设为`-1`，以禁用锁等待超时回滚，该事务会一直等待自己所需要的锁被其他事务或者会话释放。

You can also use operation-specific timeout by setting [`maxTransactionLockRequestTimeoutMillis`](https://docs.mongodb.com/manual/reference/parameters/#param.maxTransactionLockRequestTimeoutMillis) to `-1`.

Transactions release all locks upon abort or commit.

原文链接：[MongoDB document:Transactions](https://docs.mongodb.com/manual/core/transactions/)

翻译：张锐志