# order by 执行流程

## 1 全字段排序

排序过程可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数 sort_buffer_size。
sort_buffer_size，就是 MySQL 为排序开辟的内存（sort_buffer）的大小。

如果要排序的数据量小于 sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

### 1.1 试验

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

```sql
/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

### 1.2 执行流程

1. 初始化 sort_buffer，确定放入 name、city、age 这三个字段；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id；
3. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到 city 的值不满足查询条件为止；
6. 对 sort_buffer 中的数据按照字段 name 做快速排序；
7. 按照排序结果取前 1000 行返回给客户端。

## 2 rowid 排序

全字段排序算法的不足：如果查询要返回的字段很多的话，sort_buffer 里要放的字段太多，则内存里能够同时放下的行数很少，需要分成很多个临时文件，在外部文件中进行归并排序。性能差。

rowid 排序算法改进：sort_buffer 的字段，只存要排序的列（即 name 字段）和主键 id。

### 2.1 试验

```sql
# max_length_for_sort_data，是 MySQL 中专门控制用于排序的行数据的长度的一个参数。
# 它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。
SET max_length_for_sort_data = 16;
```

### 2.2 执行流程

1. 初始化 sort_buffer，确定放入两个字段，即 name 和 id；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id；
3. 到主键 id 索引取出整行，取 name、id 这两个字段，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到不满足 city='杭州’条件为止；
6. 对 sort_buffer 中的数据按照字段 name 进行排序；
7. 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

## 3 覆盖索引

    MySQL设计思想：如果内存够，就要多利用内存，尽量减少磁盘访问。

进一步改进：根据索引的有序性，以及覆盖索引（索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。）来去掉排序动作。

### 3.1 试验

```sql 
# 构建联合索引（覆盖索引）
alter table t add index city_user_age(city, name, age);
```

```sql
explain select city, name, age from t where city='杭州' order by name limit 1000 ;
```

经过 explain 命令分析，Extra 字段中 “Using index” 表示使用了覆盖索引。