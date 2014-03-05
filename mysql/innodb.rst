关与innoDB的学习与研究
=========================

内存
------

* innoDB的内存主要由以下组成：

  + buffer poll 缓冲池

  + redo log buffer

  + additional memory poll 额外内存池


缓存池是占用内存最多的部分，存放着各种数据的缓冲。InnoDB总是将数据按照page（16k）读到缓冲池，然后按照LRU（最少使用算法）保留缓存数据。如果数据需要修改，总是首先修改缓存池中的page（即为脏页），然后按照一定的频率将缓冲池的脏页刷新（flush）到文件。
缓冲池中缓存的数据页类型有：索引页、数据页、undo页、插入缓冲（insert buffer）、自适应哈系索引、InnoDB锁信息（lock info）、数据字典信息等。

redo log buffer是redolog的缓冲，一般情况夏每一秒钟就会将缓冲刷新到文件。

额外的内存池同样很重要，InnoDB中对内存的管理通过内存堆(heap)的方式进行，buffer poll中的缓存控制对象需要从额外内存池中申请。这个值应该随着缓冲池增加而增加。

master thread
-----------------

InnoDB的主要工作实在一个单独后台线程master thread中完成的。


