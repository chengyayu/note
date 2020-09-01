# SQL 执行流程

## 1. order by

### 1.1 全字段排序

#### 1.1.1 试验

```sql
# 表定义
CREATE TABLE `t` (
    `id` int(11) NOT NULL,
    `city` varchar(16) NOT NULL,
    `name` varchar(16) NOT NULL,
    `age` int(11) NOT NULL,
    `addr` varchar(128) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `city` (`city`)
) ENGINE=InnoDB;
```

```sql
explain select city, name, age from t where city='杭州' order by name limit 1000 ;
```

经过 explain 命令分析，Extra 字段中 “Using filesort” 表示需要排序，MySQL 会给每个线程分配一块内存用于排序，称为 sort_buffer。

#### 1.1.2 执行流程

1. 初始化 sort_buffer，确定放入 name、city、age 这三个字段；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id；
3. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到 city 的值不满足查询条件为止；
6. 对 sort_buffer 中的数据按照字段 name 做快速排序；
7. 按照排序结果取前 1000 行返回给客户端。

