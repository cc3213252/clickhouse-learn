explain 20.6之后才有，执行计划，理解sql执行过程

列式存储数据库（DBMS）（Hbase也是），主要用于OLAP

OLAP 联机查询处理  
OLTP 联机事务处理

高吞吐写入能力： 用了LSM Tree的结构，用顺序append写入，能达到50-200M/s

单条query就能利用整机所有cpu，吃cpu。由于单条查询使用多cpu，不利于并发查询，
对于高qps查询业务，clickhouse不是强项。

适合宽表，不适合初始存储  
join性能不好，避免join，单表查询性能优异  

join原理，把右边表加载到内存，再匹配  