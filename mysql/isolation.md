# 隔离级别与加锁机制

## 1 事务的隔离性（Isolation）

### 1.1 多个事务同时执行的问题

- **脏读（dirty read）：** 指一个事务中访问到了另外一个事务未提交的数据。
- **不可重复读（non-repeatable read）：** 一个事务读取同一条记录2次，得到的结果不一致。
- **幻读（phantom read）：** 一个事务读取2次，得到的记录条数不一致。

### 1.2 隔离级别

    SQL 标准提供了四种事务隔离级别，不同数据库有不同实现。隔离级别越严，效率越低。
    MySQL 默认可重复读。

- **读未提交（read uncommitted）：** 一个事务还没提交时，它做的变更就能被别的事务看到。
- **读提交（read committed）：** 一个事务提交之后，它做的变更才会被其他事务看到。
- **可重复读（repeatable read）：** 一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。
- **串行化（serializable）：** 对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

----

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :---: | :---: | :---:| :---: |
| READ-UNCOMMITTED | √ | √ | √ |
| READ-COMMITTED | × | √ | √ |
| REPEATABLE-READ | × | × | √ |
| SERIALIZABLE | × | × | × |

----

### 1.3 RR 隔离级别的实现

#### 1.3.1 MVCC

![Image of MVCC](./static/mvcc.png)

每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。
在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view。

回滚日志何时删除？当系统中没有比这个回滚日志更早的 read-view 的时候。==》尽量不要使用长事务

```sql
# 查找持续时间超过 60s 的事务
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

#### 1.3.2 为解决幻读加锁机制的演化

```sql
# 数据表准备
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

# 数据准备
INSERT INTO t VALUES(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

**假设**加锁机制为：**只给 id=5 这一行加锁**，而其他行的不加锁。

![Image of MVCC](./static/huandu1.png)

❗️注意：
- 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在“当前读”下才会出现。
- 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能称为幻读。幻读仅专指“新插入的行”。

事务可见性规则来分析的话，上面这三条 SQL 语句的返回结果都没有问题。因为这三个查询都是加了 for update，都是当前读。而当前读的规则，就是要能读到所有已经提交的记录的最新值。并且，session B 和 sessionC 的两条语句，执行后就会提交，所以 Q2 和 Q3 就是应该看到这两个事务的操作效果，而且也看到了，这跟事务的可见性规则并不矛盾。

❌ 问题：
- **破坏语义**：session A 在 T1 时刻就声明了，“我要把所有 d=5 的行锁住，不准别的事务进行读写操作”。而实际上，这个语义被破坏了。

- **破坏数据一致性**：

![Image of MVCC](./static/huandu2.png)

执行后**数据库**会是什么结果？
```
t1 ---> id=5 (5,5,100)
t2 ---> id=0 (0,5,5)
t4 ---> 增加 (1,5,5)
```

**binlog**会是什么内容？
```sql
# session B
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

# session C
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

# session A
update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```
结果：(0,5,100)、(1,5,100) 和 (5,5,100)

⛽️**改进**加锁机制，**扫描过程中碰到的行，也都加上写锁**。

![Image of MVCC](./static/huandu3.png)

由于 session A 把所有的行都加了写锁，所以 session B 在执行第一个 update 语句的时候就被锁住了。需要等到 T6 时刻 session A 提交以后，session B 才能继续执行。

执行后**数据库**会是什么结果？
```
t1 ---> id=5 (5,5,100)
t2 ---> blocked
t4 ---> 增加 (1,5,5)
t6 之后 session B 执行，id=0 (0,5,5)
```

**binlog**会是什么内容？
```sql
# session C
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

# session A
update t set d=100 where d=5;/*所有d=5的行，d改成100*/

# session B
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```
❌ 问题：
- id=0 这一行，已经保持一致
- id=1 这一行，在数据库里面的结果是 (1,5,5)，而根据 binlog 的执行结果是 (1,5,100)，也就是说幻读的问题还是没有解决。

**结论：即使把所有的记录都加上锁，还是阻止不了新插入的记录**

⛽️⛽️ **再改进**：引入间隙锁，间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间。

![Image of MVCC](./static/huandu4.png)

#### 1.3.3 （目前）RR 隔离级别下真正的加锁规则

- **原则1：** 加锁的基本单位是 next-key lock。
- **原则2：** 查找过程中访问到的对象才会加锁。
- **优化1：** 唯一索引上的等值查询，next-key lock 退化为行锁。
- **优化2：** 普通索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。

**示例1**

![Image of MVCC](./static/jiasuo1.png)

这里 session A 要给索引 c 上 c=5 的这一行加上读锁。

根据原则 1，加锁单位是 next-key lock，因此会给 (0,5]加上 next-key lock。

要注意 c 是普通索引，因此仅访问 c=5 这一条记录是不能马上停下来的，需要向右遍历，查到 c=10 才放弃。

根据原则 2，访问到的都要加锁，因此要给 (5,10]加 next-key lock。

但是同时这个符合优化 2：等值判断，向右遍历，最后一个值不满足 c=5 这个等值条件，因此退化成间隙锁 (5,10)。

根据原则 2 ，只有访问到的对象才会加锁，这个查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么 session B 的 update 语句可以执行完成。

但 session C 要插入一个 (7,7,7) 的记录，就会被 session A 的间隙锁 (5,10) 锁住。需要注意，在这个例子中，lock in share mode 只锁覆盖索引，但是如果是 for update 就不一样了。 执行 for update 时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

**示例2**

![Image of MVCC](./static/jiasuo2.png)

session A 在遍历的时候，先访问第一个 c=10 的记录。同样地，根据原则 1，这里加的是 (c=5,id=5) 到 (c=10,id=10) 这个 next-key lock。

然后，session A 向右查找，直到碰到 (c=15,id=15) 这一行，循环才结束。根据优化 2，这是一个等值查询，向右查找到了不满足条件的行，所以会退化成 (c=10,id=10) 到 (c=15,id=15) 的间隙锁。

如下图所示：
![Image of MVCC](./static/jiasuo3.png)

**示例3**

![Image of MVCC](./static/jiasuo4.png)

session A 的 delete 语句加了 limit 2。你知道表 t 里 c=10 的记录其实只有两条，因此加不加 limit 2，删除的效果都是一样的，但是加锁的效果却不同。可以看到，session B 的 insert 语句执行通过了，跟之前的结果不同。

这是因为，delete 语句明确加了 limit 2 的限制，因此在遍历到 (c=10, id=30) 这一行之后，满足条件的语句已经有两条，循环就结束了。因此，索引 c 上的加锁范围就变成了从（c=5,id=5) 到（c=10,id=30) 这个前开后闭区间，

如下图所示：
![Image of MVCC](./static/jiasuo5.png)

✅ 指导意义：**在删除数据的时候尽量加 limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。**