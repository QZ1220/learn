# mysql主从复制 高可用 事务

标签（空格分隔）： mysql 高可用 事务

---

环境信息
----

虚拟机：VMware Fusion8

linux：ubuntu 16.04

docker：18.09.2

mysql：5.7

参考链接
----
https://www.jianshu.com/p/ab20e835a73f

为了方便，使用docker新建mysql服务器，构建一主一从的基于**日志点**mysql集群，集群架构图如下，网上搜的：
![此处输入图片的描述][1]

1、安装docker-ce

2、拉取mysql5.7镜像

3、启动mysql  master  注意端口是3307
```linux
docker run --name mysql -p 3307:3306 -eMYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

4、启动mysql  slave  注意端口是3308
```linux
docker run --name mysql-slave -p 3308:3306 -eMYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

5、进入mysql master所在的容器，设置相关参数

![此处输入图片的描述][2]

进入/etc/mysql，修改my.cnf文件信息：
```linux
 vim /etc/mysql/my.cnf
```

```linux
[mysqld]

server_id=100

binlog-ignore-db=mysql

log-bin=edu-mysql-bin

binlog_cache_size=1M

binlog_format=mixed

expire_logs_days=7
slave_skip_errors=1062
```
重启mysql服务器：
```linux
service mysql restart
```
上面的命令会使容器停止，再使用一次docker start命令就可以了。

重启完成以后，登录mysql创建同步数据时使用的mysql账户信息，其中的ip为宿主机的ip：
```mysql
mysql -u root -h 192.168.199.219 -P 3307 -p
```
```mysql
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';  
```

6、同样的，mysql slave也需要进行配置，同样的目录下的my.cnf文件：
```linux
[mysqld]

server_id=101

binlog-ignore-db=mysql

log-bin=edu-mysql-slave1-bin

binlog_cache_size=1M

binlog_format=mixed

expire_logs_days=7


slave_skip_errors=1062

relay_log=edu-mysql-relay-bin

log_slave_updates=1

read_only=1
```
7、连接master和slave

登录master服务器，执行show master status;结果如下：
```mysql
+----------------------+----------+--------------+------------------+-------------------+
| File                 | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------------+----------+--------------+------------------+-------------------+
| edu-mysql-bin.000002 |      154 |              | mysql            |                   |
+----------------------+----------+--------------+------------------+-------------------+
```
记录下 File 和 Position 字段的值，后面会用到。

进入slave的mysql命令环境下，执行:
```linux
change master to master_host='192.168.199.219', master_user='slave', master_password='123456', master_port=3307, master_log_file='edu-mysql-bin.000002', master_log_pos=154, master_connect_retry=30;  
```

执行如下命令，即可开启主从同步：
```java
start slave;
```

mysql主从复制的两种方式
--------------
MySQL可以通过两种方式配置主从复制，一种是通过二进制日志（binary log）的方式；另一种是通过GTID（全局事务ID）方式。

上面我们完成的主从复制，其实就是基于bin-log日志的。下面我们再来看一下基于GTID的主从复制方式。

参考链接：https://www.jianshu.com/p/63efedc95822

其实配置和上面的基于日志的类似，首先是使用docker新建两个mysql数据库。

```mysql
-- 主数据库
docker run --name mysql-slave-gtid -p 3309:3306 -eMYSQL_ROOT_PASSWORD=root -d mysql:5.7


-- 从数据库
docker run --name mysql-slave-gtid -p 3310:3306 -eMYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

修改主mysql的my.cnf配置为：
```linux
[mysqld]
server-id=200   
log-bin=mysql-bin

gtid-mode = on
enforce-gtid-consistency = on
```
重启mysql，然后创建同步数据的用户：
```mysql
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```

修改从mysql的配置：
```linux
[mysqld]
server-id=201 
log-bin=mysql-bin

gtid-mode = on
enforce-gtid-consistency = on
```

执行change master命令：
```mysql 
change master to MASTER_HOST='192.168.199.219',MASTER_USER='slave',MASTER_PASSWORD='123456',MASTER_PORT=3309,MASTER_AUTO_POSITION=1;

start slave;
```

基本配置就做完了，可以查看一下slave的状态：
```mysql
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.199.219
                  Master_User: slave
                  Master_Port: 3309
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 194
               Relay_Log_File: a6fa4106ae2b-relay-bin.000003
                Relay_Log_Pos: 407
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 194
              Relay_Log_Space: 2473
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 200
                  Master_UUID: 97ff54fb-b77e-11e9-8e35-0242ac110005
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 97ff54fb-b77e-11e9-8e35-0242ac110005:1-5
            Executed_Gtid_Set: 97ff54fb-b77e-11e9-8e35-0242ac110005:1-5
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```
注意起止的Retrieved_Gtid_Set和Executed_Gtid_Set。

可以再主mysql创建一个db或者插入一些数据进行测试。


两种主从模式的对比
---------

 - 基于日志点的复制
它出现的比较早，是mysql主从复制较早使用的一种方式。从库数据同步依赖于binlog日志master_log_file以及日志的偏移量master_log_pos。但是随着master数据量的增加，binlog日志和偏移量也在不断变化，此时如果出现slave宕机或者需要新加一个slave，如何准确的指定master_log_file和master_log_pos的值不是那么的容易。该方式兼容性较好，如老版本mysql及MariaDB。支持MMM和MHA高可用架构。可以方便的跳过错误。


 - 基于GTID的复制
 GTID也就是全局事务id，格式一般为：source_id:transaction_id。source_id表示提交（执行）当前事务的机器id，transaction_id为全局事务id。在主从复制集群中，一个事务的GTID值是唯一的。兼容性较差，不支持老版本mysql及MariaDB。仅支持MHA高可用架构。只能通过置入空事务的方式跳过错误。

![此处输入图片的描述][3]

半同步复制
-----
mysql的版同步复制，主要是为了解决主从复制延时比较高的问题（除开网络原因，一般大事务也可能造成主从复制延长较大）。

mysql5.7以后引入了半同步复制的技术，可以有效减少主从复制的延迟。

下面我们演示一下配置半同步复制的过程。我们基于上面gtid复制的集群机器做演示。

 - master端配置

在msater安装半同步复制插件：
```mysql
install plugin rpl_semi_sync_master soname 'semisync_master.so';
```
设置超时时间，以及打开master半同步复制：
```mysql
set global rpl_semi_sync_master_timeout = 500;
set global rpl_semi_sync_master_enabled=on;

show variables like 'rpl%';
```
show一下配置的结果，如下所示：
```mysql
mysql> show variables like 'rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 500        |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
```

 - slave端配置

```mysql
install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

set global rpl_semi_sync_slave_enabled=on;
```
然后需要重启一下io进程：
```mysql
mysql> stop slave io_thread;
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql>
mysql> start slave io_thread;
Query OK, 0 rows affected (0.00 sec)

mysql> show global status like 'rpl%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Rpl_semi_sync_slave_status | ON    |
+----------------------------+-------+
```

我们直观的感受一下半同步复制的过程，首先停掉slave的io进程（stop slave io_thread;），然后主库更新数据，发现主库的更新操作要等到超时时间过后才会执行成功：
![此处输入图片的描述][4]
上面的语句花了500ms才执行成功，因为超时时间设置的是500ms。

mysql事务
-------

基础准备：

mysql版本： 5.7.27

表结构：
```mysql
CREATE TABLE `trans` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `amount` int(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=latin1;
```

mysql默认隔离级别：
```mysql
mysql> show variables like 'tx_isolation';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+
```


 - 脏读

简单来说，就是同时开启两个事务，分别为事务A和事务B，事务A读取到了事务B未提交的数据。实际演示一下：


 - 不可重复读

简单来说，就是同时开启两个事务，分别为事务A和事务B，事务A堆id=1的数据进行了更新，事务B再对这个数据进行更新，那么结果是两次事务更新结构的叠加。

  - 幻读

  简单来说，就是同时开启两个事务，分别为事务A和事务B，事务A新增一条数据id=5，此时，事务B可以对这条数据进行操作。
  
  
  

mysql性能指标监控及死锁检测
----------------

**性能类指标**
| 名称        | 说明   | 
| --------   | -----  | 
| QPS    | 数据库每秒处理的请求数量 |  
| TPS        |   数据库每秒处理的事务数量   |   
| 并发数        |    数据库实例当前并行处理的会话数量    | 
| 连接数        |    连接到数据库会话的数量    | 
| 缓存命中率        |    innodb的缓存命中率   | 

**功能类指标**
| 名称        | 说明   | 
| --------   | -----  | 
| 可用性   | 数据库能否对外正常提供服务 |  
| 阻塞       |   当前是否有阻塞的会话   |   
| 死锁        |    当前shi'wu    | 
| 慢查询        |    实时慢查询监控    | 
| 主从延迟        |    数据库主从延迟时间   | 
| 主从状态        |    数据库主从复制链路是否正常   | 


**QPS监控**
```mysql
mysql> show global status where variable_name in  ('Queries','uptime');
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| Queries       | 64     |
| Uptime        | 949411 |
+---------------+--------+
2 rows in set (0.01 sec)

mysql> show global status where variable_name in  ('Queries','uptime');
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| Queries       | 65     |
| Uptime        | 949423 |
+---------------+--------+
2 rows in set (0.01 sec)

mysql> select (65-64)/(949423-949411);
+-------------------------+
| (65-64)/(949423-949411) |
+-------------------------+
|                  0.0833 |
+-------------------------+
1 row in set (0.00 sec)
```
两次取样值相减，即可得出qps的值。

类似的，可以使用
```mysql
mysql> show global status where variable_name in  ('com_update','com_insert','com_delete','uptime');
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| Com_delete    | 0      |
| Com_insert    | 5      |
| Com_update    | 1      |
| Uptime        | 949668 |
+---------------+--------+
4 rows in set (0.01 sec)
```
语句，执行两次，计算tps。

**数据库并发数**
```mysql
mysql> show global status like 'threads_running';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| Threads_running | 1     |
+-----------------+-------+
1 row in set (0.00 sec)
```

**数据库连接数**
```mysql
mysql> show global status like 'Threads_connected';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 1     |
+-------------------+-------+
1 row in set (0.00 sec)
```

**innodb缓存命中率**

当我们查询数据库的时候，innodb存储引擎，会先将数据读取到缓存中，再次查询时可以提高查询效率。

 - （Innodb_buffer_pool_read_requests-Innodb_buffer_pool_reads）/Innodb_buffer_pool_read_requests*100%
 - Innodb_buffer_pool_read_requests:从缓存池中读取的次数
 - Innodb_buffer_pool_reads：从物理磁盘读取的次数

```mysql
mysql> show global status like 'Innodb_buffer_pool_read%';
+---------------------------------------+-------+
| Variable_name                         | Value |
+---------------------------------------+-------+
| Innodb_buffer_pool_read_ahead_rnd     | 0     |
| Innodb_buffer_pool_read_ahead         | 0     |
| Innodb_buffer_pool_read_ahead_evicted | 0     |
| Innodb_buffer_pool_read_requests      | 1605  |
| Innodb_buffer_pool_reads              | 218   |
+---------------------------------------+-------+
5 rows in set (0.00 sec)

mysql> select (1605-218)/1605;
+-----------------+
| (1605-218)/1605 |
+-----------------+
|          0.8642 |
+-----------------+
1 row in set (0.00 sec)
```
通常，一个高效的mysql数据库，缓存命中率不应该低于95%


**mysql可用性**
 
 ```mysql
 {19-10-08 16:27}[ruby-2.3.7]t4f-mbp-17055:~ wangquanzhou% mysqladmin -u root -p ping
Enter password:
mysqld is alive
 ```
 
 **阻塞**
 
 可以从sys.innodb_lock_waits查询出阻塞的sql语句。
 ![此处输入图片的描述][5]
 
 为了模拟阻塞，可以开启两个事物，对同一条数据进行update，第二个事物需要等待第一个事物提交以后，才能提交。
 
 此时，我们再查询innodb_lock_waits视图，即可得到刚刚阻塞的sql。
 ```mysql
 mysql> select * from innodb_lock_waits \G;
*************************** 1. row ***************************
                wait_started: 2019-10-08 16:59:50
                    wait_age: 00:00:13
               wait_age_secs: 13
                locked_table: `db_user`.`trans`
                locked_index: PRIMARY
                 locked_type: RECORD
              waiting_trx_id: 4376
         waiting_trx_started: 2019-10-08 16:59:50
             waiting_trx_age: 00:00:13
     waiting_trx_rows_locked: 1
   waiting_trx_rows_modified: 0
                 waiting_pid: 15
               waiting_query: update trans set amount = 2000 where id=1
             waiting_lock_id: 4376:30:3:2
           waiting_lock_mode: X
             blocking_trx_id: 4375
                blocking_pid: 13
              blocking_query: NULL
            blocking_lock_id: 4375:30:3:2
          blocking_lock_mode: X
        blocking_trx_started: 2019-10-08 16:58:55
            blocking_trx_age: 00:01:08
    blocking_trx_rows_locked: 1
  blocking_trx_rows_modified: 1
     sql_kill_blocking_query: KILL QUERY 13
sql_kill_blocking_connection: KILL 13
1 row in set, 3 warnings (0.00 sec)
 ```
 **慢查询监控**
 
 ```mysql
 mysql> use information_schema
Database changed
mysql> select * from PROCESSLIST where time>30 and command<>'sleep';
Empty set (0.00 sec)
 ```
 
 

 
  
 


  [1]: https://github.com/WQZ321123/learn/blob/master/image/mysql/master-slave.jpg?raw=true
  [2]: https://github.com/WQZ321123/learn/blob/master/image/mysql/20190730235016.jpg?raw=true
  [3]: https://github.com/WQZ321123/learn/blob/master/image/mysql/GTID.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/mysql/rpl_semi.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/image/mysql/block.jpeg?raw=true