---
layout: post
title:  "PostgreSQL中的行锁"
date:   2023-07-11 21:25:00 +0800
categories: 
published: true
---


最近了解了一下postgreSql中的行锁，再此记录一下

行锁总共有4中模式，分别是``FOR UPDATE``,``FOR NO KEY UPDATE``,``FOR SHARE``,``FOR KEY SHARE``。下面依次来说道说道这几种模式

**FOR UPDATE**

通过``SELECT ... FOR UPDATE``检索到的行数据会被锁定，其他的事务对这些行无法执行``DELETE``,``UPDATE``以及上其他锁，直到本次事务结束。反过来，执行``FOR UPDATE``也会被上述其他命令block住，直到其他命令的事务结束。
在``REPEATABLE READ``或者``SERIALIZABLE``的隔离级别下使用``FOR UPDATE``会有问题: 在事务开始后，被锁定的行如果被更改，会抛出错误。可惜的是，postgreSql并没有mysql中gap lock的功能，比如如下语句事务B依然可以顺利执行(user表id为主键索引，age为唯一索引)

```sql
transation A: begin
transation A: select * from "user" where age BETWEEN 10 and 25 for update;--此处只会锁住age在10到25岁表中已有的记录
transation B: INSERT INTO "user" ("id", "age") VALUES (6, 13);--如果表中没有13岁的行记录，依然可以成功插入
transation A: COMMIT

```

**FOR NO KEY UPDATE**

说实话，最开始搞不懂这个模式是什么用处，感觉跟``FOR UPDATE``不是一个东西么？后面才知道这个模式主要用作于有外键约束的情况下，因为``FOR UPDATE``后，有外键的相关行关联的子表数据是无法进行更新的。因为子表会去获取相关行在主表的``share lock``，所以才会有这个模式，通过字面意思也能理解就是不锁外键KEY，不会阻塞``KEY SHARE``模式。因为我们现在的开发表中基本不会建外键，所以花了很长的时间才理解这种模式，先看一种``FOR UPDATE``模式下，事务B操作子表BLOCK住的情况


```sql
--parent表为主表，child表的id引用parent的id作为外键
  CREATE TABLE parent(id INT PRIMARY KEY, balance INT);
    CREATE TABLE child(id INT REFERENCES parent(id) ON DELETE CASCADE, trx_timestamp TIMESTAMP);


transation A: begin
transation A: SELECT * FROM parent WHERE id=1 FOR UPDATE;--此处只会锁住age在10到25岁表中已有的记录
transation B: INSERT INTO child VALUES(1, now());--pending，等待parent表释放id=1行上的独占锁，因为transationB 需要此外键(id=1)在表parent上的共享锁(share lock)
transation C: INSERT INTO child VALUES(2, now());-- 可以顺利执行，表parent对id=2的没有加锁
transation A: COMMIT


```

可以用以下语句查看当前表上的行锁

```sql
CREATE EXTENSION IF NOT EXISTS pgrowlocks;
SELECT * FROM pgrowlocks('table_name');
```

使用``FOR NO KEY UPDATE``后，对非key的字段进行更新是不会block住的
```sql

transation A: begin
transation A: SELECT * FROM parent WHERE id=1 FOR NO KEY UPDATE;
transation B: INSERT INTO child VALUES(1, now());--顺利执行
transation C: INSERT INTO child VALUES(2, now());--顺利执行
transation A: COMMIT

```

**FOR SHARE**

共享锁 , 和``FOR UPDATE``不一样的是它不是独占锁，其他事务也可以拿``FOR SHARE``锁，主要是避免在执行的过程过数据被``DELETED``掉



```sql

transation A: begin
transation B: begin

transation A: SELECT * FROM parent WHERE id=1 FOR SHARE; 
transation B: SELECT * FROM parent WHERE id=1 FOR SHARE; -- 可以执行，不会阻塞
transation C: UPDATE parent set balance = balance + 100; -- 对锁定的行更新依然会阻塞!

transation A: COMMIT
transation B: COMMIT

```


**FOR KEY SHARE**

行为类似于``FOR SHARE``，对``FOR UPDATE``会阻塞，其他模式包括自身下都不会阻塞。而且期间也可以更新非key字段

```sql
transation A: begin
transation B: begin

transation A: SELECT * FROM parent WHERE id=1 FOR SHARE; 
transation B: SELECT * FROM parent WHERE id=1 FOR SHARE; -- 可以执行，不会阻塞
transation C: UPDATE parent set balance = balance + 100; -- 可以执行，不会阻塞

transation A: COMMIT
transation B: COMMIT

```

说实话，现在的接触的业务都不会用外键。所以对我来说，实际上有用的是``FOR UPDATE``和``FOR KEY SHARE``，后者也都用得少，现在都是软删除，用不上此模式。

因为行锁是没有限制的，一条语句可以涉及到N多行，所以PG并不会在内存中记录任何行锁的信息。但是锁行可能会导致磁盘写入。``FOR UPDATE``会标记锁定的行并且写入磁盘。

最后放下4中锁模式的兼容性,:x: 表示互相不兼容: 



 Requested Lock Mode   | FOR KEY SHARE  | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE 
- | - | - | - | -
 FOR KEY SHARE  |  |  |  | :x: 
 FOR SHARE  |  |  | :x:  | :x: 
 FOR NO KEY UPDATE  |  | :x:  | :x:  | :x: 
 FOR UPDATE  | :x:  | :x:  | :x:  | :x: 

_参考引用_

[1.PG官方行锁文档](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)<br>
[2.blog1](https://shiroyasha.io/selecting-for-share-and-update-in-postgresql.html)<br>
[3.blog2](https://www.cybertec-postgresql.com/en/row-locks-in-postgresql/)<br>
