# elk学习

标签（空格分隔）： rabbitMq

---

参考链接：https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/

https://www.jianshu.com/p/0962fb50ecff

我们本次的目的是搭建一个单机版的elk日志收集分析可视化组建。如下图所示：
![此处输入图片的描述][1]

os：macOs 10.14.5 

elasticsearch-5.6.4.tar.gz       

kibana-5.2.0-darwin-x86_64.tar.gz

logstash-5.6.3.tar.gz

整体来说还是比较简单的，基本都是解压以后，直接启动就可以了。

解压启动elasticsearch
-----------------

```linux
tar -zxvf elasticsearch-5.6.4.tar.gz

./elasticsearch-5.6.4/bin/elasticsearch
```

![此处输入图片的描述][2]

解压启动logstash-5.6.3
------------------
```linux
cd logstash-5.6.3/config/
vim logstash.conf

输入如下内容：
input {
     file {
        type => "log"
        path => "/Users/wangquanzhou/IdeaProjects/micro-service/log/*.log"
        start_position => "beginning"
    }
}
output {
  stdout {
   codec => rubydebug { }
  }
  elasticsearch {
    hosts => "127.0.0.1"
    index => "log-%{+YYYY.MM.dd}"
  }
}
```

然后启动应用，同样是bin下  ./logstash  启动

启动完成以后，logstash会去配置里指定的目录下收集所有的日志，如下图所示：
![此处输入图片的描述][3]

最后，启动kibana服务
-------------
```linux
tar -zxvf kibana-5.2.0-darwin-x86_64.tar.gz

./kibana-5.2.0-darwin-x86_64/bin/kibana
```
![此处输入图片的描述][4]

访问kibana的页面，http://localhost:5601，做如下配置：
![此处输入图片的描述][5]

然后就可以查看日志了：
![此处输入图片的描述][6]

Elasticsearch原理学习
-----------------



  [1]: https://github.com/Audi-A7/learn/blob/master/image/elk/elk.png?raw=true
  [2]: https://github.com/Audi-A7/learn/blob/master/image/elk/elastic.png?raw=true
  [3]: https://github.com/Audi-A7/learn/blob/master/image/elk/logstash.png?raw=true
  [4]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana.png?raw=true
  [5]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana_config.png?raw=true
  [6]: https://github.com/Audi-A7/learn/blob/master/image/elk/kibana_front_page.png?raw=true