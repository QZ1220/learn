# Spring源码阅读

标签（空格分隔）： spring 源码

---

本文主要针对springBoot、springCloud的相关组件的源码阅读及个人理解。

参考链接
----

 1. https://blog.csdn.net/woshilijiuyi/article/details/82219585
 2. https://blog.csdn.net/qq_35119422/article/details/81559410

目录
--

 1. springboot启动过程
 2. springboot的starter
 3. 自定义starter
 4. eureka源码
 5. feign源码
 6. hystrix源码

eureka源码分析
----------

各个服务组件是如何将自己注册到eureka上的呢？

简单来说就是想eureka发送REST请求将自己注册到Eureka Server上的。

那么，这些注册信息又是如何在Eureka上保存的呢？

Eureka内部通过，使用双层的Map结构来保存服务的注册信息。第一层key是服务名，通过spring.application.name配置，第二层key是具体的实例名称（比如，一个服务可以启动多个实例），通过spring.instance.hostname设置。

服务注册到eureka上以后，如何告诉eureka自己还”活着“呢？避免eureka将自己剔除呢？

服务于eureka会通过心跳机制，来告诉eureka，自己还”活着“。这也叫”服务续约“，关于续约，有下面两个重要的参数：
```java
spring.instance.lease-renewal-interval-in-seconds=30
spring.instance.lease-expiration-duration-in-seconds=90
```
前者定义了服务多久eureka通信一次，后者告诉eureka多久将无心跳的服务踢下线。上面的30和90秒都是默认时间，也就是说，不手动配置，这两个参数也是生效的，只不过取默认值。


 
