---
layout: post
title:  "python slots初探"
date:   2017-12-07 20:27:54
categories: python
tags: python
---

* content
{:toc}



## 1. __slots__的用法


### 1.1 基本用法
之前学习python的时候，知道使用__slots__能够节省内存，然而却没有在实际项目中使用过，而且也不清楚为什么能够节省内存？能够节省多少内存？记忆总是那么脆弱，那么干脆来个彻底的探索，并记录之。
首先，我们看看__slots__的基础用法：
```python
class A(object):
    __slots__ = ['name', 'attr']
    def __init__(self, name, attr):
        self.name = name
        self.attr = attr
```
在这里，我们定义了一个类A，它有两个属性`name`和`attr`，这样，在A的实例中我们就可以使用`name`和`attr`这两个属性了，但是使用一个没有包含在`__slots__`中的属性就会出错，例如：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-RCHAOXPx-1592356304329)(http://op1pzxins.bkt.clouddn.com/slotsuse.png)]

那么，到底能省多少内存呢？我们用实例来测量下。
### 1.2 内存测量工具
要测量内存使用，就需要工具。[**ipython_memeory_usage**](https://github.com/ianozsvald/ipython_memory_usage)是IPython下的实时监测每一行命令所使用的内存的工具，它可以帮助你了解每一个命令用的内存以及花费的时间。（github的readme说它不支持python 2.7，但经我实测一些简单功能还是支持的，更多的没有测试过，为保险起见，可以装python 3版本的，但是如果没有安装python 3，只是简单使用装Python 2版本应该也可以，暂时没有发现问题，如果电脑同时装有python2和3,在Windows下，可以使用 `py -2`命令和`py -3`命令分别进入python2和python3，同理，用pip install 和pip3 install分别安装python2和3的第三方库）。

在安装ipython_memory_usage之前，需要首先安装[**Memory Profiler**](https://github.com/pythonprofilers/memory_profiler)。直接通过pip install安装会出错，需要到github上(https://github.com/pythonprofilers/memory_profiler)下载安装包，然后通过python setup.py install进行安装。它是一个监测进程中内存消耗的模块，使用方法如下：
```python
from memory_profiler import profile

@profile
def my_func():
    a = [1] * (10 ** 6)
    b = [2] * (2 * 10 ** 7)
    del b
    return a
```
假设将其保存为example.py，则运行`python example.py`，可以得到类似下面的结果：
```
Line #    Mem usage  Increment   Line Contents
==============================================
     3                           @profile
     4      5.97 MB    0.00 MB   def my_func():
     5     13.61 MB    7.64 MB       a = [1] * (10 ** 6)
     6    166.20 MB  152.59 MB       b = [2] * (2 * 10 ** 7)
     7     13.61 MB -152.59 MB       del b
     8     13.61 MB    0.00 MB       return a
```
每一行的Mem Usage表示当前一共用的内存，Increment标明了这一行使用的内存大小。

安装完Memeory Profiler，我们便可以到ipython_memory_usage的github(https://github.com/ianozsvald/ipython_memory_usage)上下载安装包并进行安装，安装完之后，便可以简单的使用它进行python命令的内存与时间的测量，如下图所示：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-OzCjMoRs-1592356304331)(http://op1pzxins.bkt.clouddn.com/test_memory_use.png)]

从图中我们可以看到，import numpy这个库消耗了14.4766M内存，而生成一个10000000维的数组消耗了76.6484M内存。

### 1.3 `_slots__`内存使用和存取速度实测
#### 1.3.1 内存使用测试
最后，我们终于可以进行我们的__slots__内存测试实验了，如下图：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-dSsc3BMS-1592356304333)(http://op1pzxins.bkt.clouddn.com/123.png)]

从实验结果可以看到，对于有两个属性的实例，生成1M个实例，使用__slots__消耗了49.6250M内存，而没有使用__slots__消耗了185.9219M内存，使用__slots__的内存消耗大概为不使用__slots__的26.69%，内存节省量还是很可观的。
#### 1.3.2 存取速度测试
使用`__slots__`不但能够节省内存，也能提高存取属性的效率，下面用一个例子验证：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-2WyfvOkW-1592356304335)(http://op1pzxins.bkt.clouddn.com/sloted_timeit.png)]

在本人电脑上(python 2.7+windows),测试速度有10%左右的提升。
## 2. \__slots__使用注意事项
虽然使用__slots__能够**节省内存和存取属性的时间**，但是也有很多要注意的地方。

### 2.1 要使用__slots__，类必须继承自object,而且为了不生成__dict__，继承体系中的类必须定义__slots__
如下图所示，我们定义了三个类A、B、C，B和C继承自A，A定义了__slots__属性`attr`，B定义了一个空的__slots__，而C没有定义__slots__，ob和oc分别是B和C的实例。

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-WLBK1YX4-1592356304337)(http://op1pzxins.bkt.clouddn.com/slots_21_1.png)]

接下来，我们分别我ob和oc赋予属性`attr2`，可以看到，为ob赋予属性时提示没有属性attr2，而为oc赋予属性成功了：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-5izhRKde-1592356304338)(http://op1pzxins.bkt.clouddn.com/slots_21_2.png)]

我们查看ob和oc的全部属性，发现oc中生成了__dict__:

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-K0xulfCw-1592356304340)(http://op1pzxins.bkt.clouddn.com/slots_21_3.png)]

查看oc的__dict__:
```
In [14]: oc.__dict__
Out [14]: {'attr2': 'cc'}
```
所以即使父类定义了__slots__，子类要不生成__dict__，也要定义空的__slots__(如果不想引入新的属性的话)。
那么，如果没有继承自object会怎么样呢？我们定义一个具有__slots__的旧类D，然后对它进行实例化：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-qKrdIJpo-1592356304341)(http://op1pzxins.bkt.clouddn.com/slots_21_42.png)]

可以发现,`__slots__`并没有起限制作用.

### 2.2 使用__slots__的多继承问题
由于`__slots__`的实现不是简单的列表或者字典，多个父类的非空`__slots__`不能直接合并，所以使用时会报错（即使多个父类的非空`__slots__`是相同的）。

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-1s68rcRD-1592356304342)(http://op1pzxins.bkt.clouddn.com/slots__22_1.png)]

如果要使用多继承，那就要使用多个空`__slots__`的父类，这是关于使用`__slots__`多继承的唯一办法。

### 2.3 不要只在生成大量对象的实例时使用`__slots__`
`collections`模块的抽象基类虽然不能实例化，但是它们也声明了`__slots__`，为什么呢？因为如果用户希望继承自`collections`中的基类的类不要创建`__dict__`和`__weakref`，那么父类也必须没有这些属性，所以抽象基类要定义`__slots__`，以避免不必要的空间使用。

## 3. 一探究竟——`__slots__`为什么能够加快属性访问速度和减少内存消耗？
在1.3节我们经过实验验证，使用`__slots__`能够极大的节省内存，并且存取属性的速率也有所提升，这是为什么呢？《Learning Python》在第38章 Managed Attributes中的Descriptors中有讲到，python的`property`和`slots`都是用descriptor实现的，那么我们就从descriptor(描述符)讲起。

### 3.1 python descriptor(描述符)
在Python中，描述符协议使得我们能够截取特定属性的存取和删除，描述符是一个单独的类，它被赋值给一个**类属性**，这样就能够截取对这个类属性的获取、赋值和删除了。
一个描述符应该是这样的:
```python
class Descriptor(object):
    ""docstring goes here"""
    def __get__(self, instance, owner):...
    def __set__(self, instance, value):...
    def _delete__(self, instance):...
```
一个有**其中任意一个方法**的类就被当做描述符。这里的self是描述符的实例，instarnce是描述符实例被赋予的那个类的实例，owner是描述符实例被赋予的那个类。
```python
class Name(object):
	"""name descriptor docs"""
	def __get__(self, instance, owner):
		print 'fetch...'
		return instance._name

	def __set__(self, instance, value):
		print 'change...'
		instance._name = value

	def __delete__(self, instance):
		print 'remove...'
		del instance._name


class Person(object):
	def __init__(self, name):
		self._name = name

	name = Name()


if __name__ == '__main__':
	bob = Person("Bob Smith")
	print bob.name
	bob.name = 'Robert Smith'
	print bob.name
	del bob.name
```
输出是这样的：
```
C:\Python27\python.exe D:/LearningPython/test.py
fetch...
Bob Smith
change...
fetch...
Robert Smith
remove...
```

### 3.2 用纯python实现slots
现在，我们用纯python写一个简单的`__slots__`的实现。这里用到了元类(metaclass)，有关元类，stack overflow上有个很热的帖子[What is a metaclass in Python?](https://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python)，或者有人翻译成了中文版[深刻理解Python中的元类](http://blog.jobbole.com/21351/).
```python
class Member(object):
	# 定义描述器实现slots属性的查找
	def __init__(self, i):
		self.i = i

	def __get__(self, instance, owner):
		return instance._slotvalues[self.i]

	def __set__(self, instance, value):
		instance._slotvalues[self.i] = value


class Type(type):
	# 使用元类实现slots
	def __new__(cls, name, bases, attrs):
		slots = attrs.get('_slots_')
		if slots:
			for i, slot in enumerate(slots):
				attrs[slot] = Member(i)
			attrs['_slotvalues'] = [None] * len(slots)

		return type.__new__(cls, name, bases, attrs)


class Object(object):
	__metaclass__ = Type


class A(Object):
	_slots_ = 'x', 'y'


if __name__ == '__main__':
	a = A()
	print dir(A)
	a.x = 10
	print a.x
```
在CPython中，当一个A类定义了`__slots__=('x', 'y')`,A.x就是一个有`__get__`和`__set__`方法的`member_descriptor`，并且在每个实例中可以通过直接访问内存（direct memory access）获得。所以访问A.__dict__和A.x的速度是相近的。

在上面的例子中，我们用纯python实现了一个等价的slots,当一个元类看到`_slots_`的定义，就创建相应的描述符属性，并为实例创建一个_slotvalues列表。

这个例子与CPython不同的是：
1. 例子中`_slotvalues`是一个存储在类对象外部的列表，而在Cpython中它与实例对象存储在一起，可以通过直接访问内存获得。相应地， `member_descriptor`也不是存在外部列表，而同样可以通过直接访问内存获得。
2. 默认情况下，__new__方法会为每个实例创建一个字典__dict__来存储实例的属性，但如果定义了`__slots__`，`__new__`方法就不会再创建这个字典，以及`__weakref__`,但我们的例子没有实现这个功能。

### 3.3 更快的属性访问

默认情况下，访问一个实例的熟悉是通过访问`__dict__`来实现的，例如访问a.x就相当于访问`a.__dict__['x']`，可以简单的理解为四步：
1. `a.x` 2. `a.__dict__` 3. `a.__dict__[x]` 4. 结果
而定义了`__slots__`的类会为每个属性创建一个描述器，访问属性时就直接调用这个描述器，可以简单的理解为三步：
1. `b.x` 2. `member descriptor` 3. 结果
访问`__dict__`和访问描述器速度是相近的，所以通过`__dict__`访问相当于多了一步字典访问（哈希函数）的消耗。所以使用`__slots__`的类的属性访问速度会稍微快点，1.3.2的实验结果也验证了这点。

### 3.4 减少内存消耗
虽然使用`__slots__`能够稍微提高属性访问速度，但是节省内存才是它最大的用途。python 的字典本质上是个哈希表，它是一种以空间换时间的数据结果，而且为了解决冲突，python会在字典使用量超过2/3时，会进行2-4倍的扩容。而slots使用了描述符，只会为属性存储分配必要的内存，所以能够大幅减少内存的使用。