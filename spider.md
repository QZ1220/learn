# spider

标签（空格分隔）： spider

---

爬虫相关学习，之前写过一个简单的爬虫，爬取一个网站的图片。但是当时没做笔记，现在记得也不是很清楚，花了点时间重新学习了下。

### 首先是安装python
其实，爬虫不一定要python写，你喜欢的话，啥语言都可以。只是python这块现成封装的包比较多。

https://www.python.org/downloads/release/python-390/

下载适合自己操作系统的安装包。
![python](./image/spider/install_python.jpg)

安装过程，一路next，记得勾选将python路径添加到path，后续可以省去配置path。

### 然后是安装pip

注意，这一步不是必须的，pip的[官网](https://pip.pypa.io/en/stable/installing/)也写的很明白。

```html
pip is already installed if you are using Python 2 >=2.7.9 or Python 3 >=3.4 downloaded from python.org or if you are working in a Virtual Environment created by virtualenv or pyvenv. Just make sure to upgrade pip.
```
否则，就按照教程一步步安装即可。

### 然后是写一个简单的爬虫

```python
# coding=utf-8
import urllib.request
import re


# 定义一个方法，把整个页面下载下来
def getHtml(url):
    page = urllib.request.urlopen(url)  # 打开网页
    html = page.read()  # 读取 URL上面的数据
    return html  # 返回内容


# 再定义一个方法，筛选页面中想要的元素，通过正则表达式的匹配
def getimage(html):
    reg = r'src="(.+?\.jpg)" pic_ext'  # 定义一个正则表达式
    # re.compile() 把正则表达式编译成一个正则表达式对象
    imagere = re.compile(reg)
    # 　re.findall() 方法读取html 中包含 imgre（正则表达式）的数据。
    imagerelist = re.findall(imagere, str(html))
    # 遍历图片
    x = 0
    for imageurl in imagerelist:
        # 百度这块对图片链接做了特殊处理：部分字符使用了16进制表示，需要特殊处理，这里我懒得去处理了，这只是一个示例
        # 这里的核心是用到了urllib.urlretrieve(),方法，直接将远程数据下载到本地
        urllib.request.urlretrieve(imageurl, '%s.jpg' % x)
        x = x + 1


# 调用getHtml 传入一个网址
ht = getHtml("http://tieba.baidu.com/p/2460150866")
# 调用getimage ，拿到图片
getimage(ht)
print("拉取成功...")
```

