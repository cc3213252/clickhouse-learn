比MergeTree多了去重功能，去重只会在合并过程中出现。

create table t_order_rmt(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time Datetime
)engine = ReplacingMergeTree(create_time)
    partition by toYYYYMMDD(create_time)
    primary key(id)
    order by(id, sku_id);

insert into t_order_rmt values
(101, 'sku_001', 1000.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 11:00:00'),
(102, 'sku_004', 2500.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 12000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 600.00, '2020-06-02 12:00:00');

插入6条实际变成4条，进行了一次合并，再次插入，发现没有合并  
optimize table t_order_rmt final;  
之后就合并了，保留时间最新的一条，时间一样则保留后插入的  

是分区内去重的