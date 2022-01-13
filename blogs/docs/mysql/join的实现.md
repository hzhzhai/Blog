[Nested-Loop&emsp;Join](#Nested-Loop&emsp;Join)  
[优化Join速度](#优化Join速度)  
[Join的工作流程](#Join的工作流程)  

### Nested-Loop Join
在Mysql中，使用Nested-Loop Join(嵌套循环连接)的算法思想去优化join，举个例子：
```mysql
select * from t1 inner join t2 on t1.id=t2.tid
```
- t1称为外层表，也可称为驱动表
- t2称为内层表，也可称为被驱动表

伪代码表示：
```java
List<Row> result = new ArrayList<>();
for(Row r1 in List<Row> t1){
	for(Row r2 in List<Row> t2){
		if(r1.id = r2.tid){
			result.add(r1.join(r2));
		}
	}
}
```
Nested-Loop Join具体有3种实现算法：

**(1)Simple Nested-Loop Join(SNLJ，简单嵌套循环连接)**  
简单嵌套循环连接实际上就是简单粗暴的嵌套循环。  
如果table1有1万条数据，table2有1万条数据，那么数据比较的次数=1万*1万=1亿次。这种查询效率会非常慢，实现伪代码就跟上面的一样。

因此Mysql继续优化，衍生出Index Nested-Loop Join、Block Nested-Loop Join两种NLJ算法。  
在执行join查询时mysql会根据情况选择两种之一进行join查询。  

**(2)Index Nested-Loop Join(INLJ，索引嵌套循环连接)**  
索引嵌套循环连接是基于索引进行连接的算法。  
索引是基于内层表的，外层表直接与内层表索引进行匹配，避免和内层表的每条记录进行比较，从而利用索引的查询减少了对内层表的匹配次数，极大的提升了join的性能。  
原来的匹配次数 = 外层表行数 * 内层表行数 ==>> 优化后的匹配次数 = 外层表的行数 * 内层表索引的高度

伪代码实现：
```java
List<Row> result = new ArrayList<>();
for(Row r1 in List<Row> t1){
   if(Row r2 = searchBTree(t1.id) != null){
	 result.add(r1.join(r2));
   }
}
```
使用场景：只有内层表join的列有索引时，才能用到Index Nested-Loop Join进行连接。  
使用Index Nested-Loop Join算法时SQL的EXPLAIN结果extral列是Using index。  
由于用到索引，如果索引是辅助索引而且返回的数据还包括内层表的其他数据，则会回内层表查询数据，多了一些IO操作。  

**(2)Block Nested-Loop Join(INLJ，缓存块嵌套循环连接)**  
当有时候Join字段没法使用索引的时候，那样就不能用Index Nested-Loop Join，默认就会使用Block Nested-Loop Join。

原理：  
Block Nested-Loop Join一次性缓存多条数据，把参与查询的列缓存到Join Buffer里；  
然后拿join buffer里的数据批量与内层表的数据进行匹配，从而减少了内层循环的次数(遍历一次内层表就可以批量匹配一次Join Buffer里面的外层表数据)。

伪代码表示：
```java
select * from t1 inner join t2 on t1.tid=t2.tid
假设字段tid在t1和t2表中都没有建立索引，那么查找时就不能使用索引了。  
采用Block Nested-Loop Join算法就是每次从驱动表t1中加载一部分数据行到内存缓冲区Join Buffer 中来，然后对t2表进行全表扫描。  
扫描时每次拿t2表中的数据行与Join Buffer中的数据进行匹配，匹配完成就添加到结果集。
所以全表扫描的次数 = 驱动表t1的行数 / Join Buffer的大小。  

因为Join Buffer是内存缓冲区，在内存中进行元素比较是比较快的，而对t2表进行全表扫描是磁盘Io，是比较慢的，所以应该是尽可能减少全表扫描的次数。  
因此优化的方式一般是增大Join Buffer的大小，或者是选取数据量小的表作为驱动表，这样可以减少全表扫描的次数，减少磁盘IO。
  
List<Row> result = new ArrayList<>();
//可以把subList理解为每次从t1表中取出，加载到join buufer的那一部分数据
for( List<Row> subList in List<Row> t1){
	for(Row r2 in List<Row> t2){
		if(subList.contains(r2.tid){
			result.add(r1.join(r2));
		}
	}
}
```
使用Block Nested-Loop Join算法时，SQL的EXPLAIN结果extral列是Using join buffer。  
什么是Join Buffer？  
- Join Buffer会缓存所有参与查询的列而不是只有Join的列。
- 可以调整join_buffer_size缓存大小。join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G，之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。  
- 使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置：block_nested_loop，默认为on(开启)。

**在选择Join算法时会有优先级：Index Nested-LoopJoin > Block Nested-Loop Join > Simple Nested-Loop Join**

### 优化Join速度
- 用小结果集驱动大结果集，减少外层循环的数据量  
如果小结果集和大结果集连接的列都是索引列，mysql在内连接时也会选择用小结果集驱动大结果集，因为索引查询的成本是比较固定的，这时候外层的循环越少，join的速度便越快。  
- 为匹配的条件增加索引  
争取使用Index Nested-Loop Join减少内层表的循环次数。  
- 增大join buffer size的大小  
当使用BNLJ时，一次缓存的数据越多，那么外层表循环的次数就越少。  
- 减少不必要的字段查询  
当用到BNLJ时，字段越少，join buffer所缓存的数据就越多，外层表的循环次数就越少；  
当用到INLJ时，如果可以不回表查询，即利用到覆盖索引，则可能可以提升速度。（未经验证，只是一个推论）
- 排序时尽量使用驱动表中的字段
因为如果使用的是非驱动表中的字段进行排序，需要对循环查询的合并结果(临时表)进行排序，比较耗时。使用Explain时会发现出现Using temporary。

### Join的工作流程
full outer join：会包含两个表不满足条件的行  
left join：会包含左边的表不满足条件的行，一般会使用左边的表作为驱动表  
right join：会包含右边的表不满足条件的行，一般会使用右边的表作为驱动表  
inner join：只包含满足条件的行  
cross join：从表A中循环取出每一条记录去表B匹配，cross join后面不能跟on，只能跟where

**exits和in，join的区别**  
exists是拿外表作为驱动表，外表的数据做循环，每次循环去内表中查询数据，适用内表比较大的情况。  
例如
```mysql
select * from t1 where t1.tid exists (select t2.tid from t2)
```
转换为伪代码为：
```java
//就是t1作为驱动表
List<Row> result = new ArrayList<>();
for(Row r1 in List<Row> t1){
	for(Row r2 in List<Row> t2){
		if(r1.id = r2.tid){
			result.add(r1.join(r2));
		}
	}
}
```
而in的话正好相反，是内表作为驱动表，内表的数据做循环，每次循环去外表查询数据，适合内表比较小的情况。
```mysql
select * from A where cc in (select cc from B) 
-- 效率低，用到了A表上cc列的索引；

select * from A where exists(select cc from B where cc=A.cc) 
-- 效率高，用到了B表上cc列的索引。
```

join的实现其实是先从一个表中找出所有行(或者根据where子句查出符合条件的行)，然后去下一个表中循环寻找匹配的行，依次下去直到找到所有匹配的行。  
使用join不会去创建临时表，使用in的话会创建临时表，销毁临时表。

所以不管是in子查询，exists子查询还是join连接查询，底层的实现原理都是一样的，本质上是没有任何区别的，关键的点在关联表的顺序。  
如果是join连接查询，MySQL会自动调整表之间的关联顺序，选择最好的一种关联方式。和上面in和exists比较的结论一样，小表驱动大表才是最优的选择方式。

**not in和not exists的区别**   
如果查询语句使用了not in，那么内外表都进行全表扫描，没有用到索引；  
而not exists的子查询依然能用到表上的索引。所以无论那个表大，用not exists都比not in要快。
