+++
isCJKLanguage = true
title = "数据库查询时使用`for update`"
description = "数据库查询时使用`for update`"
keywords = ["DB", "mysql", "for update"]
date = 2023-11-24T16:14:27+08:00
authors = ["木章永"]
tags = ["DB"]
categories = ["DB"]
+++

数据库开启事务后，使用select语句并不会对查询到的数据加锁，当两个线程需要查询相同的数据，并且需要根据数据进行修改时，可能发生A线程和B线程都查询到数据，然后A线程根据结果对数据进行修改，而B线程还是根据旧的数据进行判断然后更新数据，将导致B线程的结果覆盖掉A线程的结果

解决问题的方式是要在事务中，对查询到的数据进行加锁（独占锁），一旦查询到数据后，在事务提交之前，不允许查询或修改该数据

Mysql/MariaDB 提供了for update来处理这种请情况

使用方式：`select * from tb_recall_info where id = 45 for update;`

通过增加for update 语句，mysql会对查询到的数据加独占锁

> InnoDB supports row-level locking. Selected rows can be locked using LOCK IN SHARE MODE or FOR UPDATE. In both cases, a lock is acquired on the rows read by the query, and it will be released when the current transaction is committed.
> 
> The FOR UPDATE clause of SELECT applies only when autocommit is set to 0 or the SELECT is enclosed in a transaction. A lock is acquired on the rows, and other transactions are prevented from writing the rows, acquire locks, and from reading them (unless their isolation level is READ UNCOMMITTED).
>
>If autocommit is set to 1, the LOCK IN SHARE MODE and FOR UPDATE clauses have no effect in InnoDB. For non-transactional storage engines like MyISAM and ARIA, a table level lock will be taken even if autocommit is set to 1.
>
>If the isolation level is set to SERIALIZABLE, all plain SELECT statements are converted to SELECT ... LOCK IN SHARE MODE.

更多资料参考：[https://mariadb.com/kb/en/for-update/](https://mariadb.com/kb/en/for-update/)

查询的条件的字段如果是有索引，加的是行级锁，如果没有索引，加的是表级锁

在实际的使用中，需要注意避免死锁的发生，如果Mysql检查到死锁，会返回失败，此时需要回滚事务