# count(*) 和 count(1) 有什么区别？哪个性能最好

![image-20260305175947835](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305175947835.png)



## 一、哪种 count 性能最好？

![image-20260305180007342](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305180007342.png)

### 1. count() 是什么？

在通过 count 函数统计有多少个记录时，MySQL 的 server 层会维护一个名叫 count 的计数变量。

count() 是一个聚合函数，该函数作用是**统计符合查询条件的记录中，函数指定的参数不为 NULL 的记录有多少个**。

```sql
select count(name) from t_order;
```

是统计「 t_order 表中，name 字段不为 NULL 的记录」有多少个

```sql
select count(1) from t_order;
```

这条语句是在统计 t_order 表中有多少个记录。



### 2. count(主键字段) 执行过程是怎样的？

如果表里只有主键索引，没有二级索引时，那么，InnoDB 循环遍历聚簇索引，将读取到的记录返回给server 层，**然后读取记录中的 id 值，就会 id 值判断是否为NULL**，如果不为 NULL，就将 count 变量加 1。

![image-20260305180639646](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305180639646.png)

如果表里有二级索引时，InnoDB 循环遍历的对象就不是聚簇索引，而是二级索引。

![image-20260305180701553](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305180701553.png)

因为相同数量的**二级索引记录可以比聚簇索引记录占用更少的存储空间**，所以二级索引树比聚簇索引树小，这样**遍历二级索引的 I/O 成本比遍历聚簇索引的 I/O 成本小**，因此「优化器」优先选择的是二级索引



### 3. count(1) 执行过程是怎样的？

用下面这条语句作为例子：

```sql
select count(1) from t_order;
```

如果表里只有主键索引，没有二级索引时

![image-20260305181026146](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305181026146.png)

那么，InnoDB 循环遍历聚簇索引（主键索引），将读取到的记录返回给 server 层，**但是不会读取记录中的任何字段的值**，因为 count 函数的参数是 1，不是字段，所以不需要读取记录中的字段值。参数 1 很明显并不是 NULL，因此 server 层每从 InnoDB 读取到一条记录就将 count 变量加 1

count(1) 相比 count(主键字段) 少一个步骤不需要读取记录中的字段值（也没有判断非空），所以通常会说**count(1) 执行效率会比 count(主键字段) 高一点**。

如果表里有二级索引时，InnoDB 循环遍历的对象就二级索引了

![image-20260305181247772](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260305181247772.png)



### 4. count(*) 执行过程是怎样的？

**count(`\*`) 其实等于 count(`0`)**，也就是说，当你使用 count(`*`) 时，MySQL 会将 `*` 参数转化为参数0来处理

所以，**count(\*) 执行过程跟 count(1) 执行过程基本一样的**，性能没有什么差异

![image-20260306110657199](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260306110657199.png)

而且 MySQL 会对 count(*) 和 count(1) 有个优化，如果有多个二级索引的时候，优化器会使用key_len 最小的二级索引进行扫描。没有二级索引的时候，才会采用主键索引来进行统计。



### 5. count(字段) 执行过程是怎样的？

ount(字段) 的执行效率相比前面的count(1)、 count(*)、 count(主键字段) 执行效率是最差的。

```sql
// name不是索引，普通字段
select count(name) from t_order;
```

对于这个查询来说，会采用全表扫描的方式来计数，执行效率是比较差的。



### 小结

![image-20260306113815623](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260306113815623.png)

count(1)、 count(*)、 count(主键字段)在执行的时候，如果表里存在二级索引，优化器就会选择二级索引进行扫描

所以如果要执行 count(1)、 count(*)、 count(主键字段) 时，尽量在数据表上**建立二级索引**，这样**相比于扫描主键索引效率会高一些**。

**不要使用 count(字段)** 来统计记录个数，因为它的**效率是最差**的。如果你非要统计表中该字段不为 NULL 的记录个数，建议给这个字段建立一个二级索引



## 二、为什么要通过遍历的方式来计数？

前面的案例都是基于 Innodb 存储引擎来说明的，在 MyISAM 存储引擎里，执行 count 函数的方式不一样

因为每张 MyISAM 的数据表都有一个 meta 信息有存储了row_count值，使用 MyISAM 引擎时，直接读取 row_count 值就是**没有任何查询条件下**的 count(*)函数的执行结果，只需要 O(1)复杂度，查询速度要明显快于 InnoDB

而当**带上 where 条件语句之后，MyISAM 跟 InnoDB 就没有区别了**

-----

InnoDB 支持事务，为了保证数据的一致性，它使用了 MVCC（多版本并发控制）

- **MyISAM 不支持事务**，所以它不需要处理“谁可见、谁不可见”的问题，行数就是全局统一的。

- **InnoDB 的设计目标是高性能事务处理**，它牺牲了 `count(*)` 的即时性，换取了极高的并发写入能力。



## 三、如何优化 count(*)？

如果对一张大表经常用 count(*) 来做统计，其实是很不好的。面对大表的记录统计，我们有没有什么其他更好的办法呢？

### 第一种，近似值

如果你的业务对于统计个数不需要很精确，比如搜索引擎在搜索关键词的时候，给出的搜索结果条数是一个大概值。

这时，我们就可以使用 show table status 或者 explain 命令来表进行估算。

执行 explain 命令效率是很高的，因为它并不会真正的去查询

![image-20260306122259853](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260306122259853.png)

### 第二种，额外表保存计数值

如果是想精确的获取表的记录总数，我们可以将这个计数值保存到单独的一张计数表中。

当我们在数据表插入一条记录的同时，将计数表中的计数字段 + 1。也就是说，在新增和删除操作时，我们需要额外维护这个计数表。