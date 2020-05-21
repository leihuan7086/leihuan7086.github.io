---
published: false
title: 缓存与数据库一致性探讨
date: 2020-05-15
category: 数据库
tag: 缓存 数据库
description: 如何保证缓存与数据库一致性？网上有N多关于此类的文章，但是没有一个可以给出最终最优解。关于缓存与数据库的一致性方案，没有最好的，只有最合适的。本文仅用于记录个人对缓存与数据一致性的一些理解。
---



## Cache Aside Pattern（旁路缓存）

### Cache Aside Pattern 是什么？

Cache Aside Pattern 是一种数据库缓存方案，目的是为了保证缓存与数据库的一致性，Cache Aside Pattern 提倡：

**a）读请求**

1）先读cache，再读db

2）如果cache hit，则直接返回数据

3）如果cache miss，则访问db，并将数据set回缓存

**b）写请求**

1）先写db，再删除cache

2）下一次读请求时，执行a）

**Q1：Cache Aside Pattern 为何提倡“先写db，再删除cache”？**

A1：高并发读写数据时，如果“先删除cache，再写db”，可能会出现：写线程删cache-读线程读cache-读线程读db-读线程写cache-写线程写db，此时会出现缓存与数据库不一致。Cache Aside Pattern适用于所有操作都顺利进行的场景。

**Q2：“先写db，再删除cache”有什么问题？**

A2：如果事务的原子性被破坏，写db成功，删除cache失败，Cache Aside Pattern 会导致缓存与数据库不一致。

Q3：如何解决Q2问题？

A3：“先删除cache，再写db“。如果删除cache失败，直接返回；如果“删除cache成功，写db失败“，下一次读请求会同步db到cache，缓存与数据库一致。



​                       