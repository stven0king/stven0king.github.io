---
title: python2 import urllib2报错
date: 2017-02-08 21:57:00
tags: [python2,urllib1L]
categories: Python
description: "  "
---

这段时间想玩玩python网页信息爬取，在使用urllib2这个库的时候导入失败，提示信息为：
```python
im@58user:~/PythonProjects/IOTest$ python
Python 2.7.6 (default, Oct 26 2016, 20:30:19) 
[GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import urllib2
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib/python2.7/urllib2.py", line 94, in <module>
    import httplib
  File "/usr/lib/python2.7/httplib.py", line 80, in <module>
    import mimetools
  File "/usr/lib/python2.7/mimetools.py", line 6, in <module>
    import tempfile
  File "/usr/lib/python2.7/tempfile.py", line 32, in <module>
    import io as _io
  File "io.py", line 3, in <module>
    os.remove(f)
NameError: name 'f' is not defined
```

当然我仅仅时想在命令行测试一下是否能导入urllib2这个库，却报出这么一段错误日志。

猜想原因是：自己当前的执行目录中的io.py文件覆盖了python自带的io文件，并且io.py脚本自己没有写完语法有问题造成的。

记录这个错误，这让作者以后pyhon在命名文件时不是那么随意～！～！

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！