## insert

表到表的插入：  
insert into [table_name] select a,b,c from [table_name2]

## update和delete

这类操作称为Mutation查询，是很重的操作，不支持事务，不建议频繁小数据操作，建议批量变更  

删除：  
alter table t_order_smt delete where sku_id = 'sku_001';

修改：  
alter table t_order_smt update total_amount = toDecimal32(2000.00, 2) where id=102;

### Mutation操作底层原理

1、新增数据新增分区，并把旧分区打上逻辑上的失效标记  
2、触发分区合并时，删除旧数据释放磁盘空间

上面的操作会发现实际lib目录下新增了一个目录，clickhouse把原数据拷贝了一份到新目录，这样效率较低  

实现高性能update或delete的思路：  
create table A
(
a xxx,
b xxx,
c xxx,
_sign UInt8,
_Version UInt32
)
更新： 插入一条新的数据， _version + 1
      查询的时候加一个过滤条件， where version最大
删除： _sign，0表示未删除，1表示已删除
      查询时候加上一个过滤条件，where _sign=0
要增加类似合并机制，把过期数据清除

## 查询操作

支持CTE（common table express）,就是可以用with，支持join，但是要避免使用

支持窗口函数  
cube多维

### 演示rollup

从右至左去掉维度统计  
清空数据：  
alter table t_order_mt delete where 1=1;

insert into t_order_mt values
(101, 'sku_001', 1000.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 11:00:00'),
(102, 'sku_004', 2500.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 12000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 600.00, '2020-06-02 12:00:00');

select id, sku_id, sum(total_amount) from t_order_mt group by id, sku_id with rollup;

┌──id─┬─sku_id──┬─sum(total_amount)─┐
│ 102 │ sku_004 │              2500 │
│ 102 │ sku_002 │             16600 │
│ 101 │ sku_001 │              1000 │
└─────┴─────────┴───────────────────┘
┌──id─┬─sku_id─┬─sum(total_amount)─┐
│ 102 │        │             19100 │
│ 101 │        │              1000 │
└─────┴────────┴───────────────────┘
┌─id─┬─sku_id─┬─sum(total_amount)─┐
│  0 │        │             20100 │
└────┴────────┴───────────────────┘

第一个是group by id, sku_id  
第二个是group by id  
第三个是group by  

### 演示cube

从右到左去掉维度，再从左到右去掉维度统计  
select id, sku_id, sum(total_amount) from t_order_mt group by id, sku_id with cube;

┌──id─┬─sku_id──┬─sum(total_amount)─┐
│ 102 │ sku_004 │              2500 │
│ 102 │ sku_002 │             16600 │
│ 101 │ sku_001 │              1000 │
└─────┴─────────┴───────────────────┘
┌──id─┬─sku_id─┬─sum(total_amount)─┐
│ 102 │        │             19100 │
│ 101 │        │              1000 │
└─────┴────────┴───────────────────┘
┌─id─┬─sku_id──┬─sum(total_amount)─┐
│  0 │ sku_004 │              2500 │
│  0 │ sku_001 │              1000 │
│  0 │ sku_002 │             16600 │
└────┴─────────┴───────────────────┘
┌─id─┬─sku_id─┬─sum(total_amount)─┐
│  0 │        │             20100 │
└────┴────────┴───────────────────┘

### total

with total，只有两个，所有的加一个汇总的

## alter操作

新增字段：  
alter table tableName add column newcolname String after col1;

修改字段：  
alter table tableName modify column newcolname String;

删除字段：  
alter table tableName drop column newcolname;

## 导出数据

clickhouse-client --password GhTY1OeM --query "select * from t_order_mt where create_time='2020-06-01 12:00:00'" > ~/rs1.csv