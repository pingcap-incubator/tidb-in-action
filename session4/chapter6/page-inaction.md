## 6.1.4 一种高效分页批处理方案

常规的分页更新 SQL 一般使用主键或者唯一索引进行排序，这样能避免相邻两页之间出现空隙或重叠；再配合 MySQL `limit` 语法中非常好用的 `offset` 功能按固定行数拆分页面，然后把页面包装进独立的事务中，从而实现灵活的分页更新。

```
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 0,10000);
commit;
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 10000,10000);
commit;
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 20000,10000);
commit;
```
如上述 SQL 所示，该方案逻辑清晰，代码也易于编写。但是，劣势也很明显：由于需要对主键或者唯一索引进行排序，越靠后的页面参与排序的行数就会越多，相应地扫描数据过程中对 TiKV 的压力也会线性增长。这导致整体处理效率偏低，尤其当批量处理涉及的数据体量较大时，可能会占用过多计算资源，甚至引发性能问题，影响联机业务。

本节将介绍一种改进方案：首先将数据按照主键排序，然后调用窗口函数 `row_number()` 为每一行数据生成行号，接着调用聚合函数按照设置好的页面大小对行号进行分组，最终计算出每页的最小值和最大值。

这里我们假定的业务需求是，要在一小时内完成 200 万行数据的并发处理。下面我们来初始化一张表 `tmp_loan`，表结构如下所示，该表初始状态即包含约 200 万行数据。

```
MySQL [demo]> desc tmp_loan;
+-------------+-------------+------+------+---------+-------+
| Field       | Type        | Null | Key  | Default | Extra |
+-------------+-------------+------+------+---------+-------+
| serialno    | int(11)     | NO   | PRI  | NULL    |       |
| name        | varchar(40) | NO   |      |         |       |
| businesssum | int(10)     | NO   |      | 0       |       |
+-------------+-------------+------+------+---------+-------+

MySQL [demo]> select count(1) from tmp_loan;
+----------+
| count(1) |
+----------+
|  1998985 |
+----------+

MySQL [demo]> select * from tmp_loan limit 10;
+-----------+-----------+-------------+
| serialno  | name      | businesssum |
+-----------+-----------+-------------+
| 200000000 | 华碧波    |       10000 |
| 200000001 | 陶南      |       10000 |
| 200000002 | 何谷      |       10000 |
| 200000003 | 曹念      |       10000 |
| 200000004 | 潘旋千    |       10000 |
| 200000005 | 魏柔      |       10000 |
| 200000006 | 公羊      |       10000 |
| 200000007 | 司马      |       10000 |
| 200000008 | 陶之      |       10000 |
| 200000009 | 严香      |       10000 |
+-----------+-----------+-------------+
```

常规的分片方式是采用 `mod` 函数对主键取余。下面的代码演示了具体做法：

```
# ${ThreadNums} 是分片数量, 用于确定最大分片数是多少
# ${ThreadId} 是当前分片的序号, 用于确定是哪一个分片  
select serialno 
  from tmp_loan 
 where mod(substring(serialno,-3),${ThreadNums}) = ${ThreadId} 
order by serialno;

# 下面是一个具体的例子, 取出第一个分片的 serialno
MySQL [demo]> select serialno from tmp_loan where MOD(substring(serialno,-3),17) = 1 order by serialno;
+-----------+
| serialno  |
+-----------+
| 200000001 |
| 200000018 |
| 200000035 |
| 200000052 |
| 200000069 |
| 200000086 |
| ......... |
+-----------+
117942 rows in set (1.407 sec)
```

如上所示，一共把 `tmp_loan` 表分成了 17 片，每个分片约为 12 万行左右。后续可针对每一个分片执行 SQL 获取数据，并遍历结果集逐行处理；并且，多个分片可以并行处理以提升效率。从 MySQL 切换到 TiDB 后，由于 `GC lift time` 默认设置为 10 分钟，遍历结果集执行批量更新处理的时间可能超过此限制而导致 TiDB 报错 `GC life time is shorter than transaction duration`。为规避这类问题，我们通常会增大分片总数，减少每个分片的行数，以缩短单一分片的处理时间。但是，新的问题也随之而来：分片总数增多之后，需要并发处理的分片数目也必然随之增多，否则无法满足一小时内完成批处理的业务目标；当同时运行几十条 SQL，每一条都要调用 `mod` 函数扫描整表，往往会引发性能问题，影响联机业务。

改进方案的基本思路是，将数据按照主键排序后，借助窗口函数 `row_number()` 生成行号，然后调用聚合函数进行分组，计算出每一页的最大值和最小值。下面的代码演示了具体的做法：

```
MySQL [demo]> SELECT min(t.serialno) AS start_key, max(t.serialno) AS end_key, count(*) AS page_size FROM ( SELECT *, row_number () over (ORDER BY serialno) AS row_num FROM tmp_loan ) t GROUP BY floor((t.row_num - 1) / 50000) ORDER BY start_key;
+-----------+-----------+-----------+
| start_key | end_key   | page_size |
+-----------+-----------+-----------+
| 200000000 | 200050001 |     50000 |
| 200050002 | 200100007 |     50000 |
| 200100008 | 200150008 |     50000 |
| 200150009 | 200200013 |     50000 |
| 200200014 | 200250017 |     50000 |
|  ........ |.......... | ........  |
| 201900019 | 201950018 |     50000 |
| 201950019 | 201999003 |     48985 |
+-----------+-----------+-----------+
40 rows in set (1.51 sec)
```

接下来，只需要使用 `serialno between start_key and end_key` 查询每个分片的数据即可。

```
MySQL [demo]> select serialno from tmp_loan where serialno BETWEEN 200050002 and 200100007;
+-----------+
| serialno  |
+-----------+
| 200050002 |
| 200050003 |
| 200050004 |
| 200050005 |
| 200050006 |
| ......... |
+-----------+
50000 rows in set (0.070 sec)
```

相较于使用 `mod` 函数，窗口函数 `row_number()` 的优势在于分片数量较多时可有效降低 TiKV 扫描数据的压力并提高查询速度。当我们需要批量修改数据时，也可以借助上面计算好的分片信息，实现高效数据更新。

```
MySQL [demo]> update tmp_loan set businesssum = 6666 where serialno BETWEEN 200000000 and 200050001;
Query OK, 50000 rows affected (0.89 sec)
Rows matched: 50000  Changed: 50000  Warnings: 0

MySQL [demo]> select * from tmp_loan ORDER BY serialno limit 10;
+-----------+-----------+-------------+
| serialno  | name      | businesssum |
+-----------+-----------+-------------+
| 200000000 | 华碧波    |        6666 |
| 200000001 | 陶南      |        6666 |
| 200000002 | 何谷      |        6666 |
| 200000003 | 曹念      |        6666 |
| 200000004 | 潘旋千    |        6666 |
| 200000005 | 魏柔      |        6666 |
| 200000006 | 公羊      |        6666 |
| 200000007 | 司马      |        6666 |
| 200000008 | 陶之      |        6666 |
| 200000009 | 严香      |        6666 |
+-----------+-----------+-------------+
```

总体而言，改进方案由于规避了频繁的数据排序操作造成的性能损耗，从而显著改善了批量处理的效率。
