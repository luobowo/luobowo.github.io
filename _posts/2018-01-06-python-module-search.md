---
layout: post
title:  "Python模块搜索路径"
date:   2018-01-06 13:01:54
categories: python
tags: python
---

* content
{:toc}

最近在学习python的C++扩展（pybind11)，写完一个扩展模块之后，想要在自己的环境中以后都能自动导入这个模块，而不用手动去添加路径(修改sys.path)应该怎么弄？以前最开始学习Python的时候看过这块内容，然而时间长了总会记忆不清，就再回顾了一遍。




概括来说，Python的自动搜索路径是这样的：
---
1. 程序的根目录
2. PYTHONPATH环境变量设置的目录
3. 标准库的目录
4. 任何能够找到的.pth文件的内容
5. 第三方扩展的*site-package*目录

---
最终，这五个部分的拼接成为了**sys.path**，其中第一和第三、第五部分是自动定义的。  
***根目录(自动)***
Python首先在根目录搜索要导入的文件。这个根目录的入口依赖于你怎么运行代码；当你运行一个程序时，这个入口就是程序运行入口(top-level script file)文件所在的目录；当你用交互式窗口期运行代码时，这个入口就是你所在的工作目录。   
***PYTHONPATH 目录(可配置的)***  
接下来，python会搜索PYTHONPATH环境变量里列出的所有目录，因为这个搜索在标准库之前，所以要小心不要覆盖一些标准库的同名模块。  
***标准库目录(自动)***  
这个没什么好说的，pyton会自动搜寻标准库模块所在的目录。  
***.pth文件列出的目录(可配置的)***  
这是用的比较少的一个python特性。它允许用户以每行一个的方式列出搜索路径，它和PYTHONPATH环境变量的不同是会在标准库路径之后搜索；而且它是针对这个python安装的，而不是针对用户的（环境变量会随着用户的不同而不同）。 那么，.pth文件应该放在哪里呢？可以通过以下代码找到.pth文件可以放置的位置：
```
import site
site.getsitepackages()
```
在我的环境中，输出如下：
```python
['C:\\Python27', 'C:\\Python27\\lib\\site-packages']
```
***Lib/site-package目录(自动)***  
最后，python会在搜索路径上自动加上site-packages目录，这一般是第三方扩展安装的地方，一般是由**distutils**工具发布的。

#### 举例说明
讲了这么多，现在我们举个小栗子。我的python环境是windows7 + python 2.7。  
1. 首先，我们新建一个环境变量PYTHONPATH，在里面加上目录`E:\python_extensions`

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-I3QdBNB0-1592356934303)(http://op1pzxins.bkt.clouddn.com/module_search_1.png)]

2. 然后，我们在`C:\Python27`目录下新增一个add.pth文件，里面的内容是：  `E:\python_extensions2`
3. 最后，我们在`E:\python_extensions`和`E:\python_extensions2`目录下分别新建模块`test.py`和`test2.py`，里面都写了一个`test`方法。  
我们打开交互解释器，结果如下：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-q6SFtgxo-1592356934305)(http://op1pzxins.bkt.clouddn.com/module_search_2.png)]

可以看到，我们可以直接导入这两个目录下的模块了。查看sys.path:

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-Rij7dysO-1592356934306)(http://op1pzxins.bkt.clouddn.com/module_search_3.png)]

嗯，这两个路径已经自动加入sys.path变量了。


## 总结
本文简要回顾了python的自动搜索路径，以及如何配置一些搜索路径以使得python在启动时能够将某些目录加到搜索路径。当然，这些自动搜索路径随着python版本和实现的不同会有细微差别，但对于目前的使用来说已经够了。


#### 参考文献
1. 《Learning Python, 5th Edition》, Chapter 22: Modules: The Big Picture, The Modules Search Path

