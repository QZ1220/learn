# jenkins持续集成笔记

标签（空格分隔）： jenkins

---

在此输入正文

版本
--
java version "1.8.0_144"
Apache Maven 3.5.0 


下载jenkins.war包并且运行
--------------
在http://www.wakaleo.com/books/jenkins-the-definitive-guide链接上，点击下图中标红的部分：

![此处输入图片的描述][1]


下载下来的jenkins.war包可以直接使用java -jar jenkins.war命令进行启动，如果启动出现错误，大部分原因是端口的问题，可以使用如下命令指定端口（如果不指定，默认是8080）：
```java
java -jar jenkins.war --httpPort=1080
```
假设端口为默认的8080，打开浏览器http://localhost:8080，初次登录需要输入一个秘钥，页面会提示你去哪里找这个秘钥文件。

![此处输入图片的描述][2]


接下来是插件的安装，由于证券的电脑需要设置代理才能进行网络访问，所以这一步选择右下角的跳过，接下来是用户名密码的设置，直接设置就好了。

![此处输入图片的描述][3]


配置jenkins
---------
在浏览器端，使用配置的用户名和密码进行登录。

![此处输入图片的描述][4]


依次点击上图中的系统管理-》管理插件-》高级，设置代理，这样才能在“可选插件”栏进行插件的在线安装。

![此处输入图片的描述][5]


如果不想使用在线安装的方式安装插件，也可以选择离线下载，再上传到jenkins的方式。大概的步骤是：http://updates.jenkins-ci.org/download/plugins/，上下载需要安装的插件包，下载完成以后在  系统管理-》管理插件-》高级-》上传插件  选项进行插件上传，然后jenkins会安装上传的插件。
安装完成的插件在  系统管理-》管理插件-》已安装  页面可以看到。

构建一个项目
------

 1. 主页面上选择“新建item”；
 2. 输入item的名称以及选择项目风格，一般我们选择“构建一个自由风格的软件项目”；
 3. 确定以后，就可以进行具体的项目配置了；
 4. 下图中的git，设置项目的源码的git地址，构建触发器选择“Poll SCM”，设置值为“H/5 * * * *”，表示每5分钟检测一次Git地址项目是否有更新，有就进行构建，否则不做任何操作。
 
![此处输入图片的描述][6]


配置的部分可以参考《Jenkins权威指南》这本老外的书，写的还挺好的。

除了等待定时到自动构建外，我们还可以进行自动构建。如下图所示。
![此处输入图片的描述][7]


  [1]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/download.png?raw=true
  [2]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/login.png?raw=true
  [3]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/username_password.png?raw=true
  [4]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/home.png?raw=true
  [5]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/proxy.png?raw=true
  [6]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/git.png?raw=true
  [7]: https://github.com/WQZ321123/learn/blob/master/image/jenkins/mannual_build.png?raw=true