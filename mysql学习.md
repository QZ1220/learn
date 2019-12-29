# mysql学习

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
 
 这里顺便说一下information_schema.PROCESSLIST表，表结构如下：
```sql
 CREATE TEMPORARY TABLE `PROCESSLIST` (
  `ID` bigint(21) unsigned NOT NULL DEFAULT '0',
  `USER` varchar(16) NOT NULL DEFAULT '',
  `HOST` varchar(64) NOT NULL DEFAULT '',
  `DB` varchar(64) DEFAULT NULL,
  `COMMAND` varchar(16) NOT NULL DEFAULT '',
  `TIME` int(7) NOT NULL DEFAULT '0',
  `STATE` varchar(64) DEFAULT NULL,
  `INFO` longtext
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```

 - https://zhuanlan.zhihu.com/p/30743094

各个字段含义如下：

 - Id: 就是这个线程的唯一标识，当我们发现这个线程有问题的时候，可以通过 kill
   命令，加上这个Id值将这个线程杀掉。前面我们说了show processlist
   显示的信息时来自information_schema.processlist 表，所以这个Id就是这个表的主键。
 - User: 就是指启动这个线程的用户。
 - Host: 记录了发送请求的客户端的IP和端口号。通过这些信息在排查问题的时候，我们可以定位到是哪个客户端的哪个进程发送的请求。
 - DB: 当前执行的命令是在哪一个数据库上。如果没有指定数据库，则该值为 NULL 。
 - Command: 是指此刻该线程正在执行的命令。这个很复杂，下面单独解释
 - Time: 表示该线程处于当前状态的时间。
 - State: 线程的状态，和 Command 对应，下面单独解释。
 - Info: 一般记录的是线程执行的语句。默认只显示前100个字符，也就是你看到的语句可能是截断了的，要看全部信息，需要使用 show full processlist。

Command的枚举值如下所示：

 - Binlog Dump: 主节点正在将二进制日志 ，同步到从节点
 - Change User: 正在执行一个 change-user 的操作
 - Close Stmt: 正在关闭一个Prepared Statement 对象
 - Connect: 一个从节点连上了主节点
 - Connect Out: 一个从节点正在连主节点
 - Create DB: 正在执行一个create-database 的操作
 - Daemon: 服务器内部线程，而不是来自客户端的链接
 - Debug: 线程正在生成调试信息
 - Delayed Insert: 该线程是一个延迟插入的处理程序
 - Drop DB: 正在执行一个 drop-database 的操作
 - Execute: 正在执行一个 Prepared Statement
 - Fetch: 正在从Prepared Statement 中获取执行结果
 - Field List: 正在获取表的列信息
 - Init DB: 该线程正在选取一个默认的数据库
 - Kill : 正在执行 kill 语句，杀死指定线程
 - Long Data: 正在从Prepared Statement 中检索 long data
 - Ping: 正在处理 server-ping 的请求
 - Prepare: 该线程正在准备一个 Prepared Statement
 - ProcessList: 该线程正在生成服务器线程相关信息
 - Query: 该线程正在执行一个语句
 - Quit: 该线程正在退出
 - Refresh：该线程正在刷表，日志或缓存；或者在重置状态变量，或者在复制服务器信息
 - Register Slave： 正在注册从节点
 - Reset Stmt: 正在重置 prepared statement
 - Set Option: 正在设置或重置客户端的 statement-execution 选项
 - Shutdown: 正在关闭服务器
 - Sleep: 正在等待客户端向它发送执行语句
 - Statistics: 该线程正在生成 server-status 信息
 - Table Dump: 正在发送表的内容到从服务器
 - Time: Unused

各个枚举值还可以查看官方文档：https://dev.mysql.com/doc/refman/5.6/en/thread-commands.html
 
 **此外**，慢查询还可以使用如下命令进行监控：
```mysql
show status like '%slow_queries%';
```
使用如下命令可以查看与查询相关的数据库配置：
```mysql
-- show variables like '%quer%';
```
如下图所示，图中同时展示了设置相关参数的sql语句，注意设置慢查询的时间是以秒为单位的：

![此处输入图片的描述][6]
 
 **主从延迟**
 ![此处输入图片的描述][7]
基本原理，就是生成一张表，主库定时自动往表中插入数据，从库定时查询主库插入的数据是否在从库存在。

**死锁**

使用percona-toolkit 的 pt-deadlock-logger命令。

安装percona-toolkit可以参考链接：http://macappstore.org/percona-toolkit/

**Install the App**

 1. Press Command+Space and type Terminal and press enter/return key.
 2. Run in Terminal app:
```ruby
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" < /dev/null 2> /dev/null
```
and press enter/return key. 
If the screen prompts you to enter a password, please enter your Mac's user password to continue. When you type the password, it won't be displayed on screen, but the system would accept it. So just type your password and press ENTER/RETURN key. Then wait for the command to finish.
 3. Run:
```linux
brew install percona-toolkit
```

https://www.percona.com/doc/percona-toolkit/LATEST/pt-deadlock-logger.html

在dba库下创建存储死锁信息的表：
```sql
use dba;
CREATE TABLE deadlocks (
server char(20) NOT NULL,
ts timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
thread int unsigned NOT NULL,
txn_id bigint unsigned NOT NULL,
txn_time smallint unsigned NOT NULL,
user char(16) NOT NULL,
hostname char(20) NOT NULL,
ip char(15) NOT NULL, -- alternatively, ip int unsigned NOT NULL
db char(64) NOT NULL,
tbl char(64) NOT NULL,
idx char(64) NOT NULL,
lock_type char(16) NOT NULL,
lock_mode char(1) NOT NULL,
wait_hold char(1) NOT NULL,
victim tinyint unsigned NOT NULL,
query text NOT NULL,
PRIMARY KEY (server,ts,thread)
) ENGINE=InnoDB;
```

然后使用如下命令(如果已经存在死锁表，那么省去下面的--create-dest=table，否则按照pt默认的格式建表)：
```mysql
{19-10-08 17:26}[ruby-2.3.7]t4f-mbp-17055:~ wangquanzhou% pt-deadlock-logger u=root,p=root,h=127.0.0.1 \
> --create-dest=table \
> --dest u=root,p=root,h=127.0.0.1,D=dba,t=deadlocks
```

然后人为制造死锁（需要开启两个事物）：

事物A：
```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update db_user.trans set amount=amount+1 where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

事物B:
```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update db_user.trans set amount=amount+1 where id=2;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

事物A:
```sql
mysql> update db_user.trans set amount=amount-1 where id=2;
Query OK, 1 row affected (10.48 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

事物B：
```sql
mysql> update db_user.trans set amount=amount-1 where id=1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
mysql>
```

产生了死锁，查看死锁表：
```sql
mysql> select * from deadlocks;
```

另外一种方式可以通过日志记录数据库的死锁，使用如下命令开启数据库死锁日志的记录：
```sql
set persist innodb_print_all_deadlocks=on;
```



数据库设计注意事项
---------
如果需要设计一个关系型数据库，那么一般需要考虑：架构、索引、锁、语法、理论范式等方面。具体一点如下图所示：

![此处输入图片的描述][8]

其中，索引管理、锁设计是重点部分，也是一般面试关注的比较多的地方。

 
 

mysql索引结构
---------
关于索引的结构，之前以及有过[笔记][9]，这里只是补充一点，关于innodb 副索引与主索引的关系。

之前关于主索引的查询步骤还是比较清晰的，但是对于副索引如何查询还是有点疑惑，下面这张图很好的解释了查询的过程，其实副索引也是一个B+-tree的结构，只不过他的叶子节点不再存储数据，而是存储主键索引的值。

查询时，首先根据副索引，找到主键索引的值，然后再在主索引的B+-tree中进行查询。就酱。（PS：之前认为副索引存储的是指向主键索引地址的指针）

![此处输入图片的描述][10]

mysql复合索引最左匹配原则及原因
------------------
最左匹配原则，可以查看下图，当然这只是针对复合索引而言：

![此处输入图片的描述][11]
![此处输入图片的描述][12]

 
 

mysql count(id)可能不走主键索引
-----------------------
count时不走主键索引，很好理解，mysql可能会自己进行优化，因为主键索引是索引+数据的存储模式，而副索引仅仅存储主键索引的值，从读取IO来说，肯定是数据量越少读取的越快。并且，副索引和主索引都是B+-tree的结构，都可以在叶子节点进行横向扫描。
![此处输入图片的描述][13]

 
 

mysql慢查询
--------
一般定位和处理慢查询可以按照下图的步骤进行处理：

![此处输入图片的描述][14]
![此处输入图片的描述][15]

那么，如何打开和查询慢查询日志呢？可以参照下图的sql进行设置：

![此处输入图片的描述][16]


  
 


  [1]: https://github.com/WQZ321123/learn/blob/master/image/mysql/master-slave.jpg?raw=true
  [2]: https://github.com/WQZ321123/learn/blob/master/image/mysql/20190730235016.jpg?raw=true
  [3]: https://github.com/WQZ321123/learn/blob/master/image/mysql/GTID.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/mysql/rpl_semi.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/image/mysql/block.jpeg?raw=true
  [6]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223231254.png?raw=true
  [7]: https://github.com/Audi-A7/learn/blob/master/image/mysql/master-slave.png?raw=true
  [8]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223224321.png?raw=true
  [9]: https://github.com/Audi-A7/learn/blob/master/%E7%AC%94%E8%AE%B0%E6%95%B4%E7%90%86.md#hash%E7%B4%A2%E5%BC%95%E5%92%8Cbtree%E7%B4%A2%E5%BC%95%E5%92%8C%E4%BD%8D%E5%9B%BE%E7%B4%A2%E5%BC%95
  [10]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223230719.png?raw=true
  [11]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223232559.png?raw=true
  [12]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223232425.png?raw=true
  [13]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223232157.png?raw=true
  [14]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223231309.png?raw=true
  [15]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223231621.png?raw=true
  [16]: https://github.com/Audi-A7/learn/blob/master/image/mysql/QQ%E6%88%AA%E5%9B%BE20191223231254.png?raw=true