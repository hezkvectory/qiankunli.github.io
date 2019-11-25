---

layout: post
title: 《mysql技术内幕》笔记1
category: 技术
tags: Storage
keywords: mysql innodb

---

## 简介

* TOC
{:toc}

可以事先看下 [数据库的一些知识](http://qiankunli.github.io/2016/09/21/db.html)

《mysql技术内幕》 主要讲存储引擎部分。

## 内存分配

我们一般做的web系统，即或是用到了缓存数据，缓存也是分散在各个类中。那么内存集中管理的例子，或者说谈得上内存模型的例子：

1. jvm，分为不同用途的数据，堆栈等，不同区域使用不同的分配与回收逻辑
2. netty的arena，多种粒度划分现有内存，手动触发分配与回收
3. mysql的缓冲池，以页为单位，LRU（有改动）为分配回收算法

innodb引擎内存占用

1. 缓冲池
2. 重做日志缓冲
3. 额外内存池

## 索引

### 为什么是B+树——B+树的逻辑结构

查询需求：

1. 基本查询：即根据主键查询
1. 范围查询
2. 前缀匹配模糊查询
3. 排序和分页


![](/public/upload/data/mysql_bplus_tree_logic_structure.jpg)

每种查找算法都只能应用于特定的数据结构之上，例如二分查找要求被检索数据有序，而二叉树查找只能应用于二叉查找树上。

InnoDB存储B+Tree节点的方式确实非常精巧，MyISAM主要是记录了主键与对应记录地址（偏移）的映射关系。

### B+树的物理结构——按页批量读写索引数据

与操作系统操作磁盘块的逻辑基本一致，操作时以块/页为单位，而不是直接从磁盘上读取记录。事实上，哪怕是访问内存，os也从未按字节读取过数据， 全部是按批量方式读取。

||读取过程|写回磁盘||
|---|---|---|---|
|操作系统读取磁盘块|判断目标数据所在的磁盘块是否在内存（对应的缓冲块），若在则修改数据，不在则读取磁盘块到内存。|操作系统负责将缓冲块同步到磁盘上。修改的缓冲块称为脏块，操作系统会根据脏块的占比决定同步到磁盘期间，是否继续响应用户操作。|磁盘块有超级块和一般数据块的不同，不准确的说，超级块的数据加载到磁盘，刚好是一个superblock list、inode list|
|innodb引擎读取页|类似|mysql有专门线程负责将脏页（有改动的页）写入到磁盘。因为redo 日志的存在，msyql会根据redo log的剩余空间占比决定在master thread中同步还是异步 将脏页写入到磁盘。|数据页组织在一起刚好是一个B+Tree|


1. 有些文件格式（二进制文件），必须整体读取才能解析和展示，有些（主要是文本文件）则可以一部分一部分的解析和展示，比如txt。
2. 有些文件格式，在内存的数据结构与磁盘一致。有些则需要转换后写入到磁盘上。

读取记录，一次读取一页，还带来一个好处，一个页的数据结构是一个整体，支持复杂的数据结构。假设一个记录10byte，无需页offset 0~9就是第一条记录，offset10~19是第二条记录（以前我就是这么想的），即用链表而不是位置维持数据的有序性。

这其实解决了笔者一直以来的一个困惑：假设一个数据库表的主键按1~1亿分布，我先插入一条主键=1的记录，再插入主键=1000的记录，再插入主键从2~100的记录。无论采用何种索引方式，每次插入，都意味着数据文件或者索引文件中数据记录的移动，操作起磁盘来就不太高效。


一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。在mysql中，**将一个B+Tree节点的大小设为等于一个内存页，这样每个节点只需要一次I/O就可以完全载入。**B-Tree中一次检索最多需要h（树的高度）-1次I/O（根节点常驻内存）。

![](/public/upload/architecture/inner_mysql_2.png)

### B+树的物理结构——如何映射逻辑结构

|逻辑结构|物理结构|
|---|---|
|非叶子节点|索引页|
|叶子节点|数据页|

![](/public/upload/data/mysql_page_structure.png)

File Header 比较重要的几个字段

|名称|大小|描述|
|---|---|---|
|FIL_PAGE_PREV|4|该页的上一个页|
|FIL_PAGE_NEXT|4|该页的下一个页|
|FIL_PAGE_LSN|8|该页最后被修改的LSN|
|FIL_PAGE_TYPE|2|该页的类型，0x45BF为数据页|

Page Header 比较重要的几个字段

|名称|大小|描述|
|---|---|---|
|PAGE_LAST_INSERT|2|最后插入记录的位置|
|PAGE_N_RECS|2|	该页中记录（User Record）的数量|
|PAGE_LEVEL|2|	该页在索引树中位置，0000代表叶子节点|

数据页存放的是完整的每行的记录，而在非数据页的索引页中，存放的仅仅是键值及指向数据页的偏移量（页号），而不是一个完整的行级录。

![](/public/upload/data/mysql_page_tree.jpg)

Page 与 Page 之前组成双向链表（PS：貌似只有同级的页由双向链表关联），页按照主键的顺序排序，这样page 与page 可以在磁盘上隔的好远，但逻辑上是连续的。

![](/public/upload/data/mysql_bplus_tree_physical_structure.jpg)

### B+树的物理结构——页的写入与读取

**按页的方式进行内存和磁盘的交互，并且几个页组织在一起刚好是一个完整的数据结构（B+Tree）**，在内存中改变B+Tree的操作负担不大，然后有一个周期性的机制将页刷新回磁盘。

![](/public/upload/architecture/inner_mysql_3.png)

查询时

1. 树的高度一般是2~4，假设树的高度是3，则page level=2 即表示根节点。先加载根节点，然后按需加载下一个page level的页 直到叶子/数据页。
2. B+Tree索引本身并不能直接找到具体的一行记录，只能找到该行记录所在的页。
3. 数据库把页载入到内存中，然后通过Page Directory（存放着行级录在页内的相对位置）再进行二分查找

## 磁盘文件

内存管理系统将内存条编址，对每个进程看到的都是0~n。文件系统相当于将离散的磁盘存储空间编址，对每个文件看到的都是0~n。当然，进程根据进程号查找就可以，文件要根据文件名查找，因此多了一些结构。

我们常说逻辑结构、物理/存储结构

1. 存储结构是扁平的，一个文件/对象/实体的数据或连续或分散，能从offset 0到结尾找到就行。存在磁盘上通常有一定的文件格式rm、mp3，一般分为文件header和body两个部分。存在内存时，真应该也定一个数据格式。
2. 逻辑结构，逻辑结构通常不是扁平的，能够承载一定的抽象概念。比如此处的innodb存储引擎文件，物理上就是一个个页连续构成，offset 0~16kb是第一个页（假设一个页大小16kb），接着第二个页等。但页有系统页、数据页，页上有共同的segment id，那么就有了段的概念，段的功能有又不同，最终组成了一个复杂的结构。

### 物理结构

假设一个表，一个表空间文件，表名test，对应文件test.ibd，ibd就是一个文件格式，有专门的工具解析，跟rm、mp3性质上一样一样的。

首先test.ibd 被划分为一个个页，每个页有不同的功能

每个页从offset 0到结束，有一定的格式约定。页有一个重要组成部分是行记录

行记录从offset 0到结束，有一定的格式约定。

### 逻辑结构

数据部分，将一个B+Tree存在一个文件里一个个连续摆放的页上。

表空间 ==> 段 ==> 区 ==> 页。

体现在文件上，就是一个个页（看不出来段和区）。页按大小划分，这样根据页号*大小就知道页的地址。区也固定大小，分为多个页，区大小/页大小=区内页的数量。页大小可调，区大小不可调，通过两个大小维度实现固定与灵活有机统一吧。段则界定了页数据的性质，有点类似内存管理的段页机制。

## 日志点分析

[MySQL checkpoint深入分析](http://www.cnblogs.com/geaozhang/p/7341333.html)

[MySQL · 引擎特性 · InnoDB redo log漫游](http://mysql.taobao.org/monthly/2015/05/01/)



![](/public/upload/architecture/inner_mysql_1.png)

为了防止数据丢失，采用WAL，事务（具体应该是数据增删改操作）提交时，先写重做日志，再修改页。LSN(log sequence number) 用于记录日志序号，它是一个不断递增的 unsigned long类型整数。**因为写redo log是第一个要做的事儿，因此可以用lsn来做一些标记。**在 InnoDB 的日志系统中，LSN 无处不在，它既用于表示修改脏页时的日志序号，也用于记录checkpoint，通过LSN，可以具体的定位到其在redo log文件中的位置。

为了管理脏页，在 Buffer Pool 的每个instance上都维持了一个flush list，flush list 上的 page 按照修改这些 page 的LSN号进行排序。猜测：脏页刷新到磁盘时，应该也是按lsn顺序来的，不会存在较大lsn已经刷盘，而较小lsn未刷盘的情况。


|编号|lsn的某个状态值|说明|本阶段的lsn redo log所在位置|本阶段的lsn对应页的内存和硬盘一致性状态|备注|
|---|---|---|---|---|---|
|1|Log sequence number|最新日志号|||
|2|Log flushed up to |日志刷盘量|2~1:内存|2~1:不一致||
|3|Pages flushed up to|脏页刷盘量|3~2:硬盘|3~2:不一致|没找到地方显式存在|
|4|Last checkpoint at |上一次检查点的位置|4~3:硬盘|4~3:一致，此时5~3对应的redo日志已失效，可以被覆盖||
|5|0|起始lsn|5~4:硬盘|5~4:一致||

我们来回顾一下：

1. 为了保证宕机时数据不丢失，采用WAL，为了减少恢复的时间，使用了checkpoint，为了加快日志的写入速度使用了redo log buffer。磁盘上的redo log容量有限，在两个checkpoint之间，发现redo log快不够时，则刷新一定量的脏页，其对应范围的lsn redo log可以被覆盖（释放）。

2. 为了加快增删改查数据的速度，使用了缓冲池。缓冲池的容量有限，所以使用了lru。lru决定将某页从缓冲池中移除，该页恰好是脏页时，需要将数据同步到内存，连带更新Pages flushed up to。

各个环节环环相扣，像艺术品。

[[转]MySQL日志——Undo Redo](http://www.cnblogs.com/Bozh/archive/2013/03/18/2966494.html)中有一种非常贴切的描述：将redo log成为新数据（还未同步到磁盘）的备份儿，重做的时候好知道怎么做。将undo log称为老数据的备份儿，恢复的时候好知道怎么恢复。

[MySQL之Undo Log和Redo Log](https://blog.csdn.net/TheLudlows/article/details/78146777)Undo + Redo的设计主要考虑的是提升IO性能，将随机读写磁盘转换为顺序读写。虽说通过缓存数据，减少了写数据的IO。
但是却引入了新的IO，即写Redo Log的IO。

## 一些体会

1. 很多技巧是通用的，比如批量，哪怕只取一条记录，操作系统也会从磁盘上将记录所在磁盘块读到内存。对于到mysql则是一个数据页，业务系统经常一次查询多个id。其它诸如缓存等莫不如是。
2. 很多问题是通用的，这点跟第一点一脉相承。比如线程安全问题，操作系统有，mysql也有，业务系统也经常有。异步io，可以在上层如业务或mysql中伪装实现，也可以直接操作系统提供。一个问题，在哪个层次解决最合适，也是一个问题。
3. 由[互联网分层架构的本质](http://www.10tiao.com/html/249/201710/2651960455/1.html) 想到的数据在不同介质的表现形式，以mysql innodb存储引擎为例

	||表现形式|
	|---|---|
	|业务系统|一个数据对象|
	|java对象在内存|参见java对象内存模型|
	|mysql逻辑上|一行记录|
	|mysql一行记录在内存|例如compact、redundant等行记录格式|
	|mysql一页记录在内存|例如antelope、barracuda等格式|
	|mysql一页记录在文件系统|假设页大小16kb，内存数据整体复制到磁盘，地址范围page offset ~ page offset + 16kb|
	|mysql表数据在硬盘|假设启动innodb_file_per_table，对应一个xx.ibd文件|
	|一个文件在操作系统|file id|
	|一个文件在磁盘|几个磁盘块 + 部分inode块|
	
4. 上层抹不去的底层印记。磁盘天然的随机读写慢于顺序读写，迫使os、mysql进行了大量的缓冲优化。cpu缓存、java内存模型导致的变量可见性问题，事务可见性问题等。
