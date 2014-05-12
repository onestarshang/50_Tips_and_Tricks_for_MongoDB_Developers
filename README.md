###50 Tips and Tricks for MongoDB Developers
=========================================

50 Tips and Tricks for MongoDB Developers 的笔记备忘


###Tip #1: Duplicate data for speed, reference data for integrity

冗余数据保证速度，关联引用保证数据完整/一致性

内嵌文档被称为非范式化，关联引用则是范式化；

难以取舍：你不能同时保证做到快速和即时一致性；

Normalization gives us slower reads and a consistent view across all orders; 
multiple documents can atomically change (as only the reference document is actually
changing).

如何选择非范式化还是范式化主要有三个factors：

1 Are you paying a price on every read for the very rare occurrence of data changing?

2 How important is consistency?（例如实时的交易系统）

3 Do reads need to be fast?

###Tip #2: Normalize if you need to future-proof data

如果数据需要常年使用，则需要范式化

###Tip #3: Try to fetch data in a single query

目的是一次查询多条文档，而不是查询一条文档。

如果要查询的数据几乎仅用于查询（而不是更新操作），那么内嵌式文档比较适用于这种情况。

However, there are plenty of cases in which an application unit would be multiple documents, but accessible through a single query.

###Tip #4: Embed dependent fields

是否内嵌文档域需要考虑的是你想要内嵌的文档是用来做什么用的：

例如：（可能不需要内嵌）

For example, you might want to query on a tag, but only to link back to the posts with that tag, not for the tag on its own.

if only one document cares about certain information, embed the information
in that document.

如果只关乎信息本身，则内嵌文档。

###Tip #5: Embed “point-in-time” data

内嵌文档不保证时效性

（即时的一致性，我猜的）

###Tip #6: Do not embed fields that have unbound growth

像用户评论这种会无限增长、不能掌控其未来增长值得限制范围，则不要内嵌文档。

###Tip #7: Pre-populate anything you can

预分配空间

例如：

For example, suppose you are creating an application for site analytics, to see how many users visited different pages every minute over a day. We will have a pages collection,
where each document represents a 6-hour slice in time for a page. We want to store
info per minute and per hour:

`{
  "_id" : pageId,
  "start" : time,
  "visits" : {
  "minutes" : [
  [num0, num1, ..., num59],
  [num0, num1, ..., num59],
  [num0, num1, ..., num59],
  [num0, num1, ..., num59],
  [num0, num1, ..., num59],
  [num0, num1, ..., num59]
  ],
  "hours" : [num0, ..., num5]
  }
}`

###Tip #8: Preallocate space, whenever possible

尽可能的预分配存储空间

###Tip #9: Store embedded information in arrays for anonymous access

如果不能明确知道你要存储的数据的键的个数（变化的），用数组存储；

并用$elemMatch查询。

否则用内嵌文档。

###Tip #10: Design documents to be self-sufficient

不要指望mongodb帮你做数据统计，尽可能多的存储不同类型的数据，统计类的或者用于展示的。

如果是统计类的，也尽量把total类型数值（算好的总数）存入文档。

例如：

`{"_id":123,
  "apples":10,
  "oranges":5
  "total":15}`

this is good enough。

###Tip #11: Prefer $-operators to JavaScript

尽可能的多的存储不同的数据（self-sufficient 自给自足）可以降低查询的复杂度。

有些不能执行的查询就要用$where语句，

As you might imagine, $where gives your queries quite a lot of power. However, it is
also slow.

好是好，就是慢。

如果发现你的查询需要大量的$where语句，那么就要考虑重新设计你的schema了。

###Tip #12: Compute aggregations as you go

能先算的尽量都先算好

###Tip #13: Write code to handle data integrity issues

写脚本来保证数据的完整性/一致性

####Consistency fixer

一致性修正

Check computations and duplicate data to make sure that everyone has consistent
values.

####Pre-populator

预分配

Create documents that will be needed in the future.

####Aggregator

内联的统计的及时更新

Keep inline aggregations up-to-date.
Other useful scripts (not strictly related to this chapter) might be:

####Schema checker

文档检查

Make sure the set of documents currently being used all have a certain set of fields,
either correcting them automatically or notifying you about incorrect ones.

####Backup job

备份

fsync, lock, and dump your database at regular intervals.

###Tip #14: Use the correct types

numbers：

排序、比较、位运算（不支持浮点）、浮点可以使用。 

Dates：

Strings：

必须存为utf-8格式或二进制

ObjectIds：

object包含了很多信息，不要使用string

###Tip #15: Override _id when you have your own simple, unique id

可以重写，但是需要保证是增长的、唯一的、范围可控的；

考虑到B-tree的边界（？）

ObjectIds have an excellent insertion order as far as the
index tree is concerned: they always are increasing, meaning they are always being
inserted at the right edge of the B-tree.

###Tip #16: Avoid using a document for _id

_id不能是文档

###Tip #17: Do not use database references

###Tip #18: Don’t use GridFS for small binary data

GridFS requires two queries: one to fetch a file’s metadata and one to fetch its contents

Thus, if you use GridFS to store small files, you are doubling the number
of queries that your application has to do. GridFS is basically a way of breaking up large
binary objects for storage in the database.

一个是请求文件头，一个是请求内容。查询成倍。

###Tip #19: Handle “seamless” failover

捕获所有的异常，因为无缝故障转移这个功能你得不到任何异常。

This is a tricky problem that is often application-dependent, so the drivers punt on the
issue. You must catch whatever exception is thrown on network errors (you should be
able to find info about how to do this in your driver’s documentation).

###Tip #20: Handle replica set failure and failover

Your application should be able to handle all of the exciting failure scenarios that could
occur with a replica set.

备份，捕获异常

###Tip #21: Minimize disk access

two functions：

1 用SSD
2 增加内存

Tip #22: Use indexes to do more with less memory

用索引，不然慢成狗

###Tip #23: Don’t always use an index

意思是说不要加不必要的索引

Every time a new record is added, removed, or updated, every index affected by the
change must be updated.

indexes can add quite a lot of overhead to writes

产生写消耗。

###Tip #24: Create indexes that cover your queries

MongoDB can do a covered index query, where it never has to follow the pointers to
documents and just returns the index’s data to the client.

如果查询的值（字段）都带索引，则mongodb就不查找文档中的别的字段，只查询带索引的。

如果你的查询只是返回部分字段，如果这些字段都是索引，那么就只会扫描索引而不是会去查询collection，当然，如果你查询的字段里不包含某字段索引，该字段依然会返回。

###Tip #25: Use compound indexes to make multiple queries fast

复合索引

###Tip #26: Create hierarchical documents for faster scans

文档分层，可以加快文档中的键值的扫描。

没有索引的时候，是按照文档-》文档中的键值依次扫描的（卧槽奇葩）

###Tip #27: AND-queries should match as little as possible as fast
as possible

“与”查询，范围由小到大：先匹配小范围的条件再匹配大范围的条件

###Tip #28: OR-queries should match as much as possible as soon
as possible

“或”查询，范围由大到小：先匹配大范围的条件再匹配小范围的条件

###Tip #29: Write to the journal for single server, replicas
for multiserver

开日志记录，多服务器，数据复制

###Tip #30: Always use replication, journaling, or both

使用备份或者开启日志记录，或者两者都要

###Tip #31: Do not depend on repair to recover data

不要指望repair来恢复数据，除非没有别的办法：

1 很慢
2 会跳过损坏的数据（不复制直接丢掉）

###Tip #32: Understand getlasterror

getlasterror指的是所有操作中最后一个出错操作的error

###Tip #33: Always use safe writes in development

可能出现问题：

What sort of things could go wrong with a write?
A write could try to push something onto a non-array field, cause a duplicate key exception
(trying to store two documents with the same value in a uniquely indexed field),
remove an _id field, or a million other user errors.

开发时要用Safe write ，这样可以知道很多错误。
