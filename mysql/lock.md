## 1 事务的隔离性（Isolation）

### 1.1 多个事务同时执行的问题

- **脏读（dirty read）**：指一个事务中访问到了另外一个事务未提交的数据。
- **不可重复读（non-repeatable read）**：一个事务读取同一条记录2次，得到的结果不一致。
- **幻读（phantom read）**：一个事务读取2次，得到的记录条数不一致。

### 1.2 隔离级别

    SQL 标准提供了四种事务隔离级别，不同数据库有不同实现。隔离级别越严，效率越低。
    MySQL 默认可重复读。

- **读未提交（read uncommitted）**：一个事务还没提交时，它做的变更就能被别的事务看到。
- **读提交（read committed）**：一个事务提交之后，它做的变更才会被其他事务看到。
- **可重复读（repeatable read）**：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。
- **串行化（serializable）**：对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

| 隔离级别 | 脏读 | 不可重复读 | 幻影读 |
| :---: | :---: | :---:| :---: |
| READ-UNCOMMITTED | √ | √ | √ |
| READ-COMMITTED | × | √ | √ |
| REPEATABLE-READ | × | × | √ |
| SERIALIZABLE | × | × | × |

----

### 1.3 隔离级别的实现

以**可重复读级别**为例

#### 1.3.1 MVCC
![Image of MVCC](./static/mvcc.png)

每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。
在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view。

回滚日志何时删除？当系统中没有比这个回滚日志更早的 read-view 的时候。==》尽量不要使用长事务

```sql
# 查找持续时间超过 60s 的事务
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```




## 加锁规则

- 原则1：加锁的基本单位是 next-key lock。
- 原则2：查找过程中访问到的对象才会加锁。
- 优化1：唯一索引上的等值查询，next-key lock 退化为行锁。
- 优化2：普通索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。


## select for update, update, insert 加锁规则是否一样？一样的！
