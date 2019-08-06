# mysql主从复制 高可用

标签（空格分隔）： mysql 高可用

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


  [1]: https://github.com/WQZ321123/learn/blob/master/image/mysql/master-slave.jpg?raw=true
  [2]: https://github.com/WQZ321123/learn/blob/master/image/mysql/20190730235016.jpg?raw=true
  [3]: https://github.com/WQZ321123/learn/blob/master/image/mysql/GTID.png?raw=true