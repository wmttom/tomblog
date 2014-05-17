ACID事务与MVCC
===============

先说说事务，事务是关系型数据库的杀手锏，原子性、一致性、隔离性、持久性 大家都背的出来，可是这也是面试中经常被问倒的地方，也是比较难理解的地方。
主要因为ACID不同人有不同的理解，用不同的方式描述。

wiki上的答案是：

- Atomicity
    Atomicity requires that each transaction is "all or nothing": if one part of the transaction fails, the entire transaction fails, and the database state is left unchanged. An atomic system must guarantee atomicity in each and every situation, including power failures, errors, and crashes. To the outside world, a committed transaction appears (by its effects on the database) to be indivisible ("atomic"), and an aborted transaction does not happen.
- Consistency
    The consistency property ensures that any transaction will bring the database from one valid state to another. Any data written to the database must be valid according to all defined rules, including but not limited to constraints, cascades, triggers, and any combination thereof. This does not guarantee correctness of the transaction in all ways the application programmer might have wanted (that is the responsibility of application-level code) but merely that any programming errors do not violate any defined rules.
- Isolation
    The isolation property ensures that the concurrent execution of transactions results in a system state that would be obtained if transactions were executed serially, i.e. one after the other. Providing isolation is the main goal of concurrency control. Depending on concurrency control method, the effects of an incomplete transaction might not even be visible to another transaction.
- Durability
    Durability means that once a transaction has been committed, it will remain so, even in the event of power loss, crashes, or errors. In a relational database, for instance, once a group of SQL statements execute, the results need to be stored permanently (even if the database crashes immediately thereafter). To defend against power loss, transactions (or their effects) must be recorded in a non-volatile memory.

我的理解：

- 原子性
    保证的事务中的操作要不一起成功，要不一起失败，没有中间态。（这是从执行过程来描述的）
- 一致性
    保证事务把数据库从一个有效状态变为另一个有效状态。（这是从结果的影响来描述的，我的理解有效，不仅是是数据库操作层面的有效，还是应用层逻辑的有效。有时只凭借数据库的约束，无法保证真实世界模型的一致性，比如转账，扣了转出人100块，给收款人存了95，这两个操作是一个原子操作，但是在现实中并不正确。一致性主要描述一个正确的事务，一定是正确有效的，在原子性的情况下，无论事务的操作成功还是失败，都不会影响数据在现实模型中的正确有效。维护一致性是事务用户的责任。）
- 隔离性
    描绘的是并发执行的事务如果串行执行，结果都是有效的。 但是实际中考虑并发问题，隔离性其实描述的是并发控制（在ANSI/ISO SQL有四种事务隔离级别，下详）
- 持久性
    事务一旦提交，就会一直有效，无论发生什么故障。（但是实际情况下持久性也是有级别的，取决于磁盘策略）

ACID只是对事务理论上的约束，从理论上只要保证ACID，一个事务就是正确的，没有问题的。但是具体到实现，还得挨个来看。如果让我们自己来实现事务，持久性取决于我们刷新数据到磁盘的策略，先不看AC，隔离性是比较难做的。因为数据库是支持并发的，资源竞争中需要靠锁来维护。在事务中的操作别人不可见，我们需要给操作的数据加排它锁，不允许其他人查看，事务完成后把锁取消。这样一来，在数据写入或修改时，无法读取，影响了并发性能。为了优化并发性能，在写入的时候不影响并发操作，大牛们搞了个方法叫MVCC并发版本控制。MVCC并没有标准实现，每个关系型数据库都有自己的实现，有的基于乐观并发控制optimistic，有的基于悲观并发控制pessimistic，这里分析的是MySQL中INNODB存储引擎MVCC的实现。

在INNODB中，会在每行数据后隐式的保存2列数据，一列是行的创建时间，一列是行的删除时间（失效时间）。
在实际操作中，存储的并不是时间，而是事务的版本号，每开启一个新事务，事务的版本号就会递增。
在可重读Repeatable reads事务隔离级别下：

    | SELECT时，读取创建版本号<=当前事务版本号，删除版本号为空或>当前事务版本号。
    | INSERT时，保存当前事务版本号为行的创建版本号
    | DELETE时，保存当前事务版本号为行的删除版本号
    | UPDATE时，插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行

| 事务的操作在事务log中，commit后才生效。
| 数据冲突通过乐观并发控制的方式解决，在死锁中，回滚持有较少行级排它锁的事务。
| 通过MVCC，可以减少锁的使用，实现非阻塞的读操作，和只锁必要行的写操作。

在ANSI/ISO SQL中定义了四种事务隔离级别：

- 未提交可读Read uncommitted：事务所有的操作都可以被其他人看到（有脏读dirty reads问题）
- 提交可读Read committed： 事务提交后就可以被看到（事务中同样的查询可能得到不同的结果，如果在两次查询中其他事务提交了新的数据，有幻读Phantoms问题）
- 可重读Repeatable reads：事务中同样的查询可以获得相同的数据（有幻读Phantoms问题，通过MVCC解决幻读）
- 串行化Serializable：顺序执行所有操作（全部加锁的操作）

| INNODB的事务隔离级别遵守着以上四种，默认事务隔离级别是可重读，其他数据库很多是提交可读。
| 只有提交可读和可重读两种隔离级别需要使用MVCC。
| 在实际使用中，未提交可读并没有带来性能的巨大提升，串行化大幅降低并发性能，这两种隔离级别很少被使用。