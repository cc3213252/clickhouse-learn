20.6之后有的
单机版clickhouse也需要起zk，否则建表出错（TODO: 单节点不需zk，多节点没副本也不需要）  

单机启动：
clickhouse start  
zk.sh start

官网建议机器128G，给clickhouse配100G， 32线程  

explain plan select arrayJoin([1,2,3,null,null]);

┌─explain───────────────────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY))                               │
│   SettingQuotaAndLimits (Set limits and quota after reading from storage) │
│     ReadFromStorage (SystemOne)                                           │
└───────────────────────────────────────────────────────────────────────────┘

一步步执行，ReadFromStorage从存储读数据  

explain select database, table, count(1) cnt from system.parts where database in ('datasets', 'system')
 group by database,table order by database,cnt desc limit 2 by database;

打开所有参数（增加执行计划的信息）：  
explain header=1, actions=1, description=1 select number from system.numbers limit 10;  
1表示打开

语法树：  
explain ast select number from system.numbers limit 10;

## 三元运算符语法优化

select number = 1 ? 'hello' : (number = 2 ? 'world' : 'atguigu') from numbers(10);
创建具有10个随机数字的临时表  

explain syntax select number = 1 ? 'hello' : (number = 2 ? 'world' : 'atguigu') from numbers(10);

开启三元运算符优化：  
set optimize_if_chain_to_multiif = 1;  

再次查看优化：  
┌─explain─────────────────────────────────────────────────────────────┐
│ SELECT multiIf(number = 1, 'hello', number = 2, 'world', 'atguigu') │
│ FROM numbers(10)                                                    │
└─────────────────────────────────────────────────────────────────────┘

记住这种用法，以后直接这样写

## pipeline

explain pipeline select sum(number) from numbers_mt(100000) group by number % 20;  
如果系统是2核，会2个线程执行  

explain pipeline header=1, graph=1 select sum(number) from numbers_mt(100000) group by number % 20;

## 老版本打印explain

clickhouse-client --send_logs_level=trace <<< "sql" > /dev/null;