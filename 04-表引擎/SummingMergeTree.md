不查询明细，只聚合汇总，预聚合（是分区聚合，最终的才有用）

create table t_order_smt(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time Datetime
)engine = SummingMergeTree(total_amount)
    partition by toYYYYMMDD(create_time)
    primary key(id)
    order by(id, sku_id);

total_amount是想聚合的列，不能是order by中的字段（必须是非维度列）    

insert into t_order_smt values
(101, 'sku_001', 1000.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 11:00:00'),
(102, 'sku_004', 2500.00, '2020-06-01 12:00:00'),
(102, 'sku_002', 2000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 12000.00, '2020-06-01 13:00:00'),
(102, 'sku_002', 600.00, '2020-06-02 12:00:00');

可以看到聚合发生了  
insert into t_order_smt values (101, 'sku_001', 2000.0, '2020-06-01 13:00:00');  
optimize table t_order_smt final;  
聚合后，时间取最早的一条（其他列按插入顺序保留第一行）  

## 开发建议

设计聚合表，唯一键值、流水号可以去掉，所有字段全部是维度、度量或时间戳  

## 总结

只有同一批次或者分区合并时才会聚合，故查询时sum还是要加  