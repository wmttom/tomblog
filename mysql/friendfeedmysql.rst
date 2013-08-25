[原创翻译]FriendFeed如何用MySQL储存无模式数据
===============================================
| @这是一篇比较老的文章，我现在的理解是使用MySQL实现了一个MongoDB，在思路上有借鉴意义。
| 原文地址:http://backchannel.org/blog/friendfeed-schemaless-mysql

背景
------

我们使用MySQL储存FriendFeed所有的数据。我们的数据库随着我们的用户基数增长而增长了许多，现在存储了超过2亿5千万（250 million）条目和数据串，这些数据来自评论和朋友列表的“喜欢”。

伴随着数据的增长，我们反复处理因为数据过快增长而带来的扩展问题。我们曾经使用了常规方法，比如使用从属读服务器和memcache，增加读取能力；使用数据库分片来提高写入的能力。然而，随着我们的发展，发现增加新功能比扩展现有系统容量更困难。

尤其在这些情况下：设计模式改变；给一个超过一两千万行的数据库增加索引，数据库一次完全锁死几小时。删除旧索引也会占用同样的时间，并且不删除他们会损害性能。因为数据库会在每次INSERT插入操作时，继续读写这些不使用的块，并把重要的块挤出内存。存在避开这些问题的非常复杂的设计，比如在从服务器上生成索引，然后对调主从服务器。但是这些方法容易出错，太过重量级，并且暗中阻止我们添加需要改变索引或设计模式的新功能。自从我们的数据库深度分片开始，与MySQL相关的功能，比如JOIN对于我们毫无价值。所以，我们决定在关系型数据库之外寻找答案。

为了使用灵活模式和快速索引特性存储数据，诞生了许多项目，比如说CouchDB。然而，似乎他们中没有一个拥有足够的信任被大型网站广泛使用。在测试中，我们运行表明，对于我们的需求，他们中没有一款足够稳定或经受考验。MySQL可以满足需求，并且从来不会损坏数据，或重复工作。我们已经认识到了它的局限性。我们喜欢MySQL用来存储，而不是关系型数据库的使用模式。

在一些考虑之后，我们决定在MySQL之上，实现一套“无模式”的存储系统，而不是完整的使用其他新型存储系统。这篇文字试图在高层次上描述系统的细节。我们很好奇其他大型网站是如何解决这个问题的，并且我们认为一些我们做的设计工作可能会对其他开发者有所帮助。

概述
--------

我们的数据库存储无模式的属性包（比如JSON或Python中的字典）。存储实体唯一需要的属性是id，一个16字节的UUID（通用唯一标识符）。实体的其他属性在数据库连接之前是不透明的。我们可以简单地通过存储新属性来改变模式。

我们通过在单独的MySQL表中存储索引，来索引这些实体的数据。如果我们想在每个实体中索引3个属性，我们将会有3个MySQL表（每一个对应一个索引）。如果我们想要停止使用一个索引，先从代码停止向这张表写入，然后根据需要在MySQL中删除这张表。如果我们想使用一个新索引，先在MySQL中为这个索引建立新表，然后运行一个程序异步填入索引，这样不会干扰我们的正常服务。

因此，相对之前我们后来会有更多的表，但是添加和删除索引非常容易。我们拥有深度优化过的程序用来填充新索引（被我们叫做"Cleaner"），因此可以在不干扰网站正常服务的情况下迅速填充索引。我们可以在一天的时间内存储新属性并建立索引，而不是一个周的时间。并且，我们不需要再交换MySQL主从服务器，或做其他提心吊胆的操作工作让其实现。

详情
-------
在MySQL中，我们的实体像这样被存储在表中::

    CREATE TABLE entities (
	added_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
	id BINARY(16) NOT NULL,
	updated TIMESTAMP NOT NULL,
	body MEDIUMBLOB,
	UNIQUE KEY (id),
	KEY (updated)
    ) ENGINE=InnoDB;

added_id列存在，因为InnoDB存储数据行完全按照主键顺序。自增主键确保新实体在旧实体之后连续地被写入磁盘，同时帮助读写操作确定位置（因为FriendFeed页面按照年代反转排序，所以新实体相对于旧实体，倾向于拥有更频繁的读取）。实体内容被使用zlib算法压缩存储，pickle化的python字典。

索引存储在分开的单独表里。为了创建一个新索引，要新建一个表，存储我们想要索引的属性，以便在所有数据库群中查找。例如，一个标准FriendFeed实体像这样::

    {
	"id": "71f0c4d2291844cca2df6f486e96e37c",
	"user_id": "f48b0440ca0c4f66991c4d5f6a078eaf",
	"feed_id": "f48b0440ca0c4f66991c4d5f6a078eaf",
	"title": "We just launched a new backend system for FriendFeed!",
	"link": "http://friendfeed.com/e/71f0c4d2-2918-44cc-a2df-6f486e96e37c",
	"published": 1235697046,
	"updated": 1235697046,
    }

我们想索引实体属性中的user_id，以便可以渲染给定用户请求的一个页面内所有实体。索引表像这样::

    CREATE TABLE index_user_id (
	user_id BINARY(16) NOT NULL,
	entity_id BINARY(16) NOT NULL UNIQUE,
	PRIMARY KEY (user_id, entity_id)
    ) ENGINE=InnoDB;

我们数据存储代替你自动维护索引，因此开启一个数据存储的实例 ，存储给定索引上文结构的实体应该这样写（python）::

    user_id_index = friendfeed.datastore.Index(
	table="index_user_id", properties=["user_id"], shard_on="user_id")
    datastore = friendfeed.datastore.DataStore(
	mysql_shards=["127.0.0.1:3306", "127.0.0.1:3307"],
	indexes=[user_id_index])

    new_entity = {
	"id": binascii.a2b_hex("71f0c4d2291844cca2df6f486e96e37c"),
	"user_id": binascii.a2b_hex("f48b0440ca0c4f66991c4d5f6a078eaf"),
	"feed_id": binascii.a2b_hex("f48b0440ca0c4f66991c4d5f6a078eaf"),
	"title": u"We just launched a new backend system for FriendFeed!",
	"link": u"http://friendfeed.com/e/71f0c4d2-2918-44cc-a2df-6f486e96e37c",
	"published": 1235697046,
	"updated": 1235697046,
    }
    datastore.put(new_entity)
    entity = datastore.get(binascii.a2b_hex("71f0c4d2291844cca2df6f486e96e37c"))
    entity = user_id_index.get_all(datastore, user_id=binascii.a2b_hex("f48b0440ca0c4f66991c4d5f6a078eaf"))

上文的索引类在素有实体中寻找user_id属性，并自动在index_user_id表中维护索引。自从我们的数据库分片以后，shard_on属性就被用来确认，索引被存储在哪一个数据库片上（在这个案例中，值为实体的ueser_id对分片数量取余）。

你可以使用索引实例查询一个索引（请参考上文user_id_index.get_all）。数据存储系统代码会在python中完成index_user_id表和实体表之间的“join”工作，通过先在所有数据库片中查询index_user_id表，拿到实体ID的列表，然后再实体表中读取这些ID。

为了添加一个新索引，例如，在link属性上建立所有，我们应该创建一个新表::

    CREATE TABLE index_link (
	link VARCHAR(735) NOT NULL,
	entity_id BINARY(16) NOT NULL UNIQUE,
	PRIMARY KEY (link, entity_id)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

我们将会改变存储系统的初始代码来增加这个新索引::

    user_id_index = friendfeed.datastore.Index(
	table="index_user_id", properties=["user_id"], shard_on="user_id")
    link_index = friendfeed.datastore.Index(
	table="index_link", properties=["link"], shard_on="link")
    datastore = friendfeed.datastore.DataStore(
	mysql_shards=["127.0.0.1:3306", "127.0.0.1:3307"],
	indexes=[user_id_index, link_index])

并且我们可以异步填充这个索引（即使在服务繁忙的时候），使用::

    ./rundatastorecleaner.py --index=index_link

一致性与原子性
---------------

自从我们的数据库开始分片，对比实体数据本身，一个实体的索引会被存储到不同数据库片上，一致性是一个问题。假设程序在写完所有索引表前崩溃将会怎样呢？

建立一个事务协议对于大部分有抱负的FriendFeed工程师是一个诱人的方案，但是我们希望保持系统尽可能的简单。我们决定这样放开约束：

- 属性包存储在主实体表中作为标准规范
- 索引可能不会反映实体的真实值

因此，我们用以下步骤向数据库写入了新的实体：

1. 使用InnoDB的ACID属性，向实体表写入实体
2. 向所有数据库片上的所有索引表，写入索引

当从索引表读取时，我们知道结果不是非常精确的（也就是，如果写入时没有完成步骤2，索引可能反映旧的属性值）。为了保证基于以上约束，我们不会返回无效的实体，我们使用索引表来确认要读取哪一个实体，但是我们会在实体中重复提交过滤查询，而不是相信索引的完整性：

1. 基于查询在所有索引表中读取entity_id
2. 根据给定的实体ID在实体表中读取实体
3. 过滤（in python）所有不与实际属性值匹配的实体

为了确保索引不失去持久性，不一致最后会被修复，我上文提到过的“Cleaner”程序，在表间不断运行，写入丢失索引并清除旧的、无效的索引。它会先处理最近更新的实体，所以在实际中索引中的不一致会被非常快的修复（在几秒之内）。

性能
--------

我们在新系统中已经对主要索引进行了非常多的优化，并且对优化结果非常满意。下面是上个月FriendFeed页面延迟的图表（我们在几天前启动了新后台，你可以看到戏剧性的下降）：

.. image:: http://d1udwvgzrtavb8.cloudfront.net/f066c739eb6ff1a5d4f3d275ac564ce70efccda5

尤其是，我们系统的延迟现在非常稳定，即使在高峰的正午时间。下面是过去24小时FriendFeed页面延迟的图表：

.. image:: http://d1udwvgzrtavb8.cloudfront.net/72a319e1cd1c16520e26fa428bed7039ecb67f6d

对比一周前的数据：

.. image:: http://d1udwvgzrtavb8.cloudfront.net/aaf78c3d130196bf0f8863fadd7b7bf41aa04bd3

到目前为止，系统真的容易完成了工作。自从我们发展了系统，已经改变了索引好多次，并且我们开始使用新模式转换最大的的MySQL表，以便于我们可以随着发展更自由的改变数据结构。
