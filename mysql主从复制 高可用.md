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







  [1]: https://github.com/WQZ321123/learn/blob/master/image/mysql/master-slave.jpg?raw=true
  [2]: https://github.com/WQZ321123/learn/blob/master/image/mysql/20190730235016.jpg?raw=true