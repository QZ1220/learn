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

为了方便，使用docker新建mysql服务器，构建一主一从的mysql集群，集群架构图如下，网上搜的：
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

5、


  [1]: https://github.com/WQZ321123/learn/blob/master/image/mysql/master-slave.jpg?raw=true