# Python+Scrapy+Selenium数据采集

我是好人，一个大大的良民。

好与坏，关键在于使用者噢！

`Scrapy`是一个常用的数据采集工具；

`Selenium`是一个浏览器自动化测试工具；

结合`Scrapy`对数据的处理机制和`Selenium`模拟真实浏览器去获取数据（如：自动化登录，自动化翻页等）。可以更好的完成采集。

## About Scrapy

`Scrapy`是开发者在网络上用于常用的数据采集工具之一，对于通过`API`获取数据我们已经司空见惯了，但是有些`WebSite`还是会为了“性能或者安全”等原因，通过某些技术手段特意避开`API`传递数据（比如页面静态化，一次性token等）。因此为了能够采集到这些数据，我们可以通过分析站点和标签结构，然后通过借助`Scrapy`采集数据。

简单的介绍了`Scrapy`框架的作用了，那具体如何帮助我们采集数据呢？一起看看`Scrapy`的结构吧：

![](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy01.png)

`Scrapy`的数据流由`Scrapy Engine`控制，流程如下：

1. `Engine`初始化，并从`Spider`获取请求。
2. 将`Request`入调度器。
3. 调度器将`Request`逐一发送给`Scrapy Engine`消费。
4. `Scrapy Engine`通过下载器中间件将请求发送给下载器。
5. 下载器将用`Request`获取的页面作为`Response`结果返回给`Scrapy Engine`。
6. `Scrapy Engine`从`Downloader`接收`Response`并发送给`Spider`处理（Spider Middleware）。
7. `Spider`处理`Response`并向`Scrapy Engine`返回`Item`。
8. `Scrapy Engine`将处理好的`Item`发送给`Item Pipeline`，同时将已处理的信号一同发送给调度器（`Scheduler`），请求下一个采集请求。

重复以上步骤处理采集请求，直至`Scheduler`没有新的`Request`。



`Scrapy`安装教程：https://doc.scrapy.org/en/latest/intro/install.html



### Scrapy 项目创建

今天就以`清博大数据`作为案例样本，完成自动化登录，自动化搜索以及数据采集。

在文件根目录执行：

```shell
scrapy startproject qingbo
```

然后进入目录 `qingbo/` 下执行：

```
scrapy genspider crawl gsdata.cn
```



得出以下目录：

```shell
qingbo/
    scrapy.cfg            # deploy configuration file

    qingbo/             # project's Python module, you'll import your code from here
        __init__.py

        items.py          # project items definition file

        middlewares.py    # 浏览器的启动和访问方式在这操作

        pipelines.py      # 处理好后的数据在这做最后处理

        settings.py       # project settings file

        spiders/          # a directory where you'll later put your spiders
            __init__.py
            crawl.py	  # 访问的连接和爬取后的数据在这里处理
```

其实如何在`Scrapy`结合`Selenium`最关键就是在`middlewares.py`

具体如何封装可以参考下这里：https://www.osgeo.cn/scrapy/topics/downloader-middleware.html?highlight=midd#module-scrapy.downloadermiddlewares



## About Selenium

`Selenium`是一款开源的自动化测试框架，通过不同的浏览器和平台对Web应用进行校验，目前支持多个语言调用，如：Python、Java、PHP等。

Selenium 测试直接在浏览器中运行，就像真实用户所做的一样，所以利用这点，可以更好的进行数据采集。



Python Selenium安装教程：https://selenium-python-zh.readthedocs.io/en/latest/installation.html



### Selenium 案例

如果没有登录态直接访问 [清博大数据的腾讯视频](http://www.gsdata.cn/rank/wxdetail?wxname=dQmBlDkSZJWq9ipnbgG59221ZQO0O0OO0O0 ) 

不出意外的话，会跳转到登录页需要登录。上面已经提到了`Selenium`的环境安装，这里就直接上代码啦：

#### 站点打开

```python
options = Options()
options.headless = False
driver = webdriver.Firefox(options=options)
driver.get('https://u.gsdata.cn/member/login')
driver.implicitly_wait(10) # 页面打开需要加载时间，所以建议加个静默等待
```

![image-20200327102210587](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy02.png)

#### 登录操作

可以发现两个tab，分别为：二维码登录、清博帐号登录。

页面已经打开了，如何到清博帐号登录的tab呢？

这里我们需要了解一下[Xpath](https://www.w3school.com.cn/xpath/xpath_syntax.asp)（XML Path Language），它是一种用来确定XML文档中某部分位置的语言。

简单的说，就是我们可以用Xpath对“清博帐号登录”这个Tab进行定位

![image-20200327111331243](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy03.png)

```python
driver.find_element_by_xpath(".//div[@class='loginModal-content']/div/a[2]").click()
```

然后定位到账号密码框，填入信息：

```python
driver.find_element_by_xpath(".//input[@name='username']").send_keys("username")
driver.find_element_by_xpath(".//input[@name='password']").send_keys("password")
```

最后点击登录按钮：

```python
driver.find_element_by_xpath(".//div/button[@class='loginform-btn']").click()
driver.implicitly_wait(5)
```

![image-20200327112059788](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy04.png)

登录成功！~

#### 查询操作

```python
driver.get('http://www.gsdata.cn/')
driver.find_element_by_xpath(".//input[@id='search_input']").send_keys("腾讯视频")
driver.find_element_by_xpath(".//button[@class='btn no btn-default fl search_wx']").click()
driver.implicitly_wait(5)
```

![image-20200327112437222](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy05.png)

在搜索后得出以下结果：

![image-20200327112546457](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy06.png)

通过`Xpath`定位到腾讯视频的`a`标签，然后点击进入腾讯视频的数据内容页：

```python
driver.find_element_by_xpath(
    ".//ul[@class='imgword-list']/li[1]/div[@class='img-word']/div[@class='word']/h1/a").click()
driver.implicitly_wait(5)
```

#### 内容页

![image-20200327115153854](https://rawcdn.githack.com/WilburXu/blog/2bf602527c2f2eda023106c50237e511e7df59c9/other/images/scrapy07.png)

到了这里，惊不惊喜？现在就可以通过`Xpath`定位并获取需要的内容进行加工啦，就不细说了。



#### 关闭操作

```python
driver.close()
```

数据获取完，如果没有其他操作了，就可以把浏览器关了。



## 总结

本章介绍了`Scrapy`和`Selenium`的基本概念和大致的使用方法，总的来说，可以帮助我们在解决一些问题的时候提供新的方案和思路。





## Reference

https://www.cnblogs.com/luozx207/p/9003214.html

https://kite.com/blog/python/web-scraping-scrapy/

https://docs.scrapy.org/en/latest/intro/tutorial.html

