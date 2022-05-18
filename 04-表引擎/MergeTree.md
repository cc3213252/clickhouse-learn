## partition by

hive的分区是用目录实现的，clickhouse也是分目录实现  
如果不填，目录就是all  

/var/lib/clickhouse/
data是数据
metadata 表结构信息   

### metadata

ATTACH TABLE _ UUID 'dff6941c-5736-4c68-8476-38bf4b776ef5' 
(
    `id` UInt32,
    `sku_id` String,
    `total_amount` Decimal(16, 2),
    `create_time` DateTime
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(create_time)
PRIMARY KEY id
ORDER BY (id, sku_id)
SETTINGS index_granularity = 8192  索引粒度，稀疏索引

### data

目录名称20200601_1_1_0，日期后跟了三个数，第一个1最小分区块编号，第二个1最大编号，第三个0合并等级,0表示没合并过  

data.bin 数据文件
data.mrk3 标记文件，方便查找，一般记偏移量   
count.txt 行数  
primary.idx 主键索引文件，加快查询效率  
minmax_create_time.idx 分区键的最大最小值  

### 演示分区合并

snipaste 这个截图工具可以桌面置顶显示图片  

再次插入之前的sql语句后，执行：  optimize table t_order_mt final; 手动合并
查看data目录可以解释 新生成一个1-3-1  最小编号1，最大编号3，合并了1次  

再插入一次sql，再合并：  
optimize table t_order_mt partition '20200601' final;

查看目录看到多了：  
20200601_1_5_2

## 主键

查建表语句： show create table t_order_mt;
稀疏索引，用二分查找

## order by

唯一一个必填项，由于是稀疏索引，必须要有序

## 二级索引

又称跳数索引

create table t_order_mt2(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time Datetime,
    INDEX a total_amount TYPE minmax GRANULARITY 5   
)engine = MergeTree
    partition by toYYYYMMDD(create_time)
    primary key(id)
    order by (id, sku_id);

GRANULARITY是设定二级索引对于一级索引粒度的粒度  

INDEX a total_amount TYPE minmax GRANULARITY 5  // 二级索引

insert into t_order_mt2 values
(101, 'sku_001', 1000.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 11:00:00'),
(102, 'sku_004', 2500.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 12000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 600.00, '2020-06-02 12:00:00');

clickhouse-client --password GhTY1OeM --send_logs_level=trace <<< 'select * from t_order_mt2 where total_amount > toDecimal32(900, 2)';

直接查看这个sql语句执行的日志

drop table t_order_mt2；

skp_idx_a.idx2
skp_idx_a.mrk3
这两个文件就是跳数索引  

## 数据TTL

过期时间

### 字段级TTL

create table t_order_mt3(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2) TTL create_time+interval 10 SECOND,
    create_time Datetime
)engine = MergeTree
    partition by toYYYYMMDD(create_time)
    primary key (id)
    order by (id, sku_id);

上面TTL设置的列不能是主键列  
测试TTL：  

insert into t_order_mt3 values
(106, 'sku_001', 1000.00, '2022-05-18 10:26:00'),
(107, 'sku_002', 2000.00, '2022-05-18 10:26:00'),
(110, 'sku_003', 600.00, '2022-05-18 10:26:00');

时间到时total_amount列数据为0

下面这句不执行也可以
optimize table t_order_mt3 final;

这里如果不为0，可能的原因是虚拟机的时区看是否设置为东8区了，clickhouse的时区是否也设置正确

### 表级TTL

alter table t_order_mt3 modify ttl create_time + interval 10 second;  
整个表数据在create_time 10秒之后丢失  

官网有数据过期移动到磁盘功能  
可以对已有表设置某一列的ttl