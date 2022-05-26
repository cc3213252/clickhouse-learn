cdc和maxwell会加锁，如果关锁，数据一致性会没有保证

不能查询的ddl语句，不是报错，而是忽略
物化MySqL的前提是：数据库里的表都是有主键的！

## mysql设置

mysql开启binlog:  
vim /usr/local/etc/my.cnf
```ini
server-id=1
log-bin=mysql-bin
binlog_format=ROW

gtid-mode=on
enforce-gtid-consistency=1 # 设置为主从强一致性

default_authentication_plugin=mysql_native_password # 为了clickhouse可以建表
```
brew services restart mysql

创建用户，并赋予权限：  
create user 'clickhouse'@'%' identified by 'root';
grant select on *.* to 'clickhouse'@'%';
grant replication client, replication slave, reload on *.* to 'clickhouse'@'%';
flush privileges;

## clickhouse设置

开启物化引擎：  
set allow_experimental_database_materialized_mysql=1;

创建复制管道：  
create database test_binlog engine = MaterializeMySQL('192.168.10.1:3306',
'tip_asset', 'clickhouse', 'root');

vagrant中虚拟机ip为192.168.10.31，则宿主机ip就为192.168.10.1  

查看clickhouse数据：  
use test_binlog;

碰到问题：  
Code: 100. DB::Exception: Received from localhost:9000. DB::Exception: Access denied for user clickhouse. (UNKNOWN_PACKET_FROM_SERVER)


    