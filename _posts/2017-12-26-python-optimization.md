---
layout: post
title:  "python 脚本通用优化技巧"
date:   2017-12-26 13:44:54
categories: python
tags: python
---

* content
{:toc}

不知不觉用python已经一年半有余，学而不思则罔，决定花些时间好好总结下python脚本中的一些通用优化技巧，让自己在工作中少走点弯路。有些优化技巧并不只限于python，为了方便，一起写在本文中。



## 1. 工欲善其事，必先利其器
在大型项目中，当我们写完代码做完基本功能后，对性能进行分析并优化是很重要的组成步骤，特别在游戏开发中，性能瓶颈是造成游戏卡顿的重要因素，从而影响游戏体验。要对代码进行优化，首先我们要知道性能瓶颈在哪里，这就需要profile工具。Python自带的profile工具有cProfile，profile和hotshot;cProfile是C扩展实现的库，而profile是纯python实现，所以我们这里介绍cProfile。
用profile分析出性能瓶颈之后，下一步就是针对这些代码进行优化；我们怎么知道所做的优化有没有效果？timeit模块在这时就派上了用场，它能对小块代码进行计时，从而给我们的优化提供客观的评判标准。

### 1.1 cProfile
#### 1.1.1 cProfile使用入门
下面通过一个例子说明cProfile的用法，这里是一个求斐波那契数列的递归函数：
```python
def fib(n):
	if n == 0:
		return 0
	elif n == 1:
		return 1
	else:
		return fib(n-1) + fib(n-2)

def fib_seq(n):
	seq = []
	if n > 0:
		seq.extend(fib_seq(n-1))
	seq.append(fib(n))
	return seq
```
有两种方法进行profile：
1. 脚本有函数入口，则可以通过命令行执行：
```python
python -m cProfile [-o output_file] [-s sort_order] LearnCprofile.py
```
2. cProfile也可以在代码中直接调用，通过字符串执行python脚本：
```python
import cProfile
cProfile.run('print fib_seq(20)', filename=None, sort='cumtime')
```
结果类似这样：
```python
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987, 1597, 2584, 4181, 6765]
         57355 function calls (65 primitive calls) in 0.013 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.013    0.013 <string>:1(<module>)
     21/1    0.000    0.000    0.013    0.013 LearnCprofile.py:17(fib_seq)
 57291/21    0.013    0.000    0.013    0.001 LearnCprofile.py:8(fib)
       20    0.000    0.000    0.000    0.000 {method 'extend' of 'list' objects}
       21    0.000    0.000    0.000    0.000 {method 'append' of 'list' objects}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```
最上方是简要信息：57355次函数调用，其中包含65次原函数调用（指不包含递归调用的次数），共消耗0.013秒。  
**Ordered by:** 默认根据filename一列排序  
**ncalls:** 函数调用次数，如果有两个数字，则后者是原函数调用次数，前者是总共的调用次数（原函数次数+递归次数）  
**tottime:** 在函数内的时间开销，不包含子函数调用  
**percall:** tottime/ncalls，这里ncalls是总共的调用次数
**cumtime:** 在函数内的时间开销，包含子函数调用  
**percall:** cumtime/ncalls 这里ncalls是原函数调用次数  
我们也可以把结果存下来，以使用其它工具查看：
```python
cProfile.run('print fib_seq(20)', filename='result.prof', sort='cumtime')
```
#### 1.1.2 profile代码块

上面的方法是运行代码段的方式，如果想要profiler一段代码，或者某些模块，则可以新建profile对象:  
pr=cProfile.Profile()
在模块开始的地方加上  
pr.enable()  
在profile结尾的地方加上  
pr.disable()  
然后就可以查看结果：  
s=open(outfile,'wb')  
sortby = 'cumulative'  
ps = pstats.Stats(pr, stream=s).sort_stats(sortby)  
ps.dump_stats(outfile+'.prof')  
ps.print_stats()  
s.close()  

#### 1.1.3 用KCacheGrind(QCacheGrind)可视化查阅结果
虽然cProfile的结果很有意义，但是它输出的基本格式并不容易一眼获取有用信息，[KCacheGrind](https://kcachegrind.github.io/html/Home.html)([QCacheGrind](https://sourceforge.net/projects/qcachegrindwin/?source=typ_redirect))提供了更高级的可视化功能。为了使用可视化功能，我们首先要把cProfile的结果转变为CacheGrind的格式，这里我们使用[pyprof2calltree](https://pypi.python.org/pypi/pyprof2calltree):
```python
pyprof2calltree [-k] [-o output_file] [-i input_file] [-r scriptfile [args]]
```
成功转换后，会生成result.prof.log文件，在这里我们用QCacheGrind打开，便可以得到直观的各函数消耗的占比，以及函数调用关系：  
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-RKZgS9Kz-1592356748177)(http://op1pzxins.bkt.clouddn.com/per_1.1.3_2.png)]
从图中可以看到，fib_seq占了总时间的99.13%，其中，递归调用了自己20次，除去子函数调用只占0.49%，说明这个函数的开销都在子函数也就是在fib上。

### 1.2 timeit模块

通过profile我们得到了程序中的瓶颈在哪里，当我们进行优化后要对性能进行衡量和比较时，timeit模块就派上了用场。timeit模块提供了一种简便的方法为小块代码进行计时，它可以从命令行调用，从交互解释器调用以及在脚本代码中进行调用。下面简单讲下从交互解释器和脚本中调用timeit的`timeit`方法和`repeat`方法的基本使用方法。

```python
timeit.timeit(stmt='pass', setup='pass', timer=<default timer>, number=1000000)
timeit.repeat(stmt='pass', setup='pass', timer=<default timer>, repeat=3, number=1000000)
```
stmt参数为要执行的语句(statement)，按字符串的形式传入要执行的代码；第二个参数setup用于构建代码环境，可以用来导入需要的模块；number指定了运行的次数；repeat函数的repeat参数指定整个实验的重复次数，它返回的是一个包含了每次实验的执行时间的列表。

#### 1.2.1 在交互解释器中调用

```python
>>> import timeit
>>> timeit.timeit('"-".join([str(n) for n in range(100)])', number=10000)
>>> 0.20875866142209829
>>> timeit.repeat('"-".join([str(n) for n in range(100)])', repeat=3, number=10000)
>>> [0.21645912014042779, 0.21566136269410663, 0.2150726083682457]
```
repeat函数可以很方便的取多次实验的结果，以便取平均值或者最大最小值。

#### 1.2.2 在脚本中调用
timeit模块同样可以在脚本中直接使用。使用方法还是调用上述的函数。但是在脚本中时需要为setup传入一条字符串形式的语句，用于构建执行环境，示例如下：
```python
def test():
	return "-".join(str(n) for n in range(100))

if __name__ == '__main__':
	import timeit
	print timeit.timeit("test()", setup="from __main__ import test", number=10000)

>>> 0.223088509676
```

## 2. python通用优化技巧

有了前面的性能测量工具（或者说方法），我们就可以从网上、书本、自己的经验或者同事的口中等等地方得到各种让python脚本跑的更快的建议，并对其进行实验看其是否有效；如果真的有效的话，那赶快用起来吧\^-^  
以下内容记录了一些python常用的能够让代码运行的更快的方法，我暂且将其称之为优化技巧吧，这里的内容并没有进行分类，可能有点混乱；对于大部分tips，都会用实验来验证其有效性。

### 2.1 对列表排序时，对于key和cmp参数，优先使用key参数;且优先使用列表内置的sort方法，而不要使用内置函数sorted

对列表进行排序是程序中很常见的操作，一般来说，我们并不需要自己去写排序方法，Python列表内置的sort方法以及python内置的sorted函数足够满足我们平常的需求；然而，看似这么不起眼的操作，不同的使用姿势也会导致较大的效率差别：
```python
>>> inport timeit
>>> L = [('b', 2), ('a', 1), ('c', 3), ('d', 4)]
>>> min(timeit.repeat('L.sort(cmp=lambda x,y:cmp(x[1],y[1]))', setup='from __main__ import L', repeat=10))
>>> 1.0088758427042421
>>> min(timeit.repeat('L.sort(key=lambda x:x[1])', setup='from __main__ import L', repeat=10))
>>> 0.9107629156605981
>>> min(timeit.repeat('sorted(L, key=lambda x:x[1])', setup='from __main__ import L', repeat=10))
>>> 1.3784661444045696
>>> min(timeit.repeat('sorted(L, cmp=lambda x,y:cmp(x[1], y[1]))', setup= 'from __main__ import L', repeat=10))
>>> 1.9223029418607211
```
从上面的结果我们可以看到，用key参数会比用cmp参数快不少，而且用列表内置的sort方法比用sorted快很多。列表内置的sort方法是**稳定的原地排序**；而sorted是非稳定排序，且只会返回一个排序后的当前对象的副本，而不会改变当前对象。
用dis模块分析python字节码，发现cmp参数要比key参数多执行6条指令。
```python
 >>> dis.dis("L.sort(cmp=lambda x,y: cmp(x[1], y[1]))")
  0 INPLACE_RSHIFT
  1 <46>
  2 POP_JUMP_IF_TRUE 29295
  5 LOAD_GLOBAL     25384 (25384)
  8 IMPORT_FROM     15728 (15728)
 11 IMPORT_NAME     28001 (28001)
 14 DELETE_GLOBAL   24932 (24932)
 17 SLICE+2
 18 SETUP_LOOP      31020 (to 31041)
 21 INPLACE_DIVIDE
 22 SLICE+2
 23 DUP_TOPX        28781
 26 STORE_SLICE+0
 27 SETUP_LOOP      12635 (to 12665)
 30 FOR_ITER         8236 (to 8269)
 33 SETUP_EXCEPT    12635 (to 12671)
 36 FOR_ITER        10537 (to 10576)

 >>> dis.dis("L.sort(key=lambda x:x[1])")
  0 INPLACE_RSHIFT
  1 <46>
  2 POP_JUMP_IF_TRUE 29295
  5 LOAD_GLOBAL     27432 (27432)
  8 LOAD_NAME       15737 (15737)
 11 IMPORT_NAME     28001 (28001)
 14 DELETE_GLOBAL   24932 (24932)
 17 SLICE+2
 18 SETUP_LOOP      30778 (to 30799)
 21 DELETE_NAME     23857 (23857)
 24 STORE_SLICE+1
```
 
###  2.2 字符串拼接
 
 在我们的印象中，字符串拼接时，用join方法比用+快，而且，一般教科书上也推荐我们用join的方法；确实，单就这两个操作来说，join确实比+快，但是join要有可迭代对象，而生成可迭代对象是要消耗时间的，**所以，字符串拼接时用+还是用join方法是有争议的，<font color=red>要看是否已经有可迭代对象:</font>**
```python
src = ("alpha", "beta", "gamma", "delta")

def test_plus():
    res = ""
    for s in src:
        res += s

def test_join1():
    li = []
    for s in src:
        li.append(s)
    res = "".join(li)

def test_join2():
    li = [s for s in src]
    res = "".join(li)

def test_join3():
    res = "".join(src)

if __name__ == '__main__':
    import timeit
    print "plus1:", timeit.timeit("test_plus()", setup="from __main__ import test_plus")
    print "join1:", timeit.timeit("test_join1()", setup="from __main__ import test_join1")
    print "join2:", timeit.timeit("test_join2()", setup="from __main__ import test_join2")
    print "join3:", timeit.timeit("test_join3()", setup="from __main__ import test_join3")
	
	
[out]:
plus1: 0.365366068032
join1: 0.690094713869
join2: 0.434651535504
join3: 0.213327494516
```
 test\_plus1使用+的方法生成字符串；test\_join1和test\_join2分别用append以及列表解析的方法生成可迭代的列表，再用join的方法生成字符串；而test\_join3假设已经有可迭代对象了，可以直接用join的方法生成字符串；最后的结果显示，速度从快到慢依次是test\_join3 > test\_plus1 > test\_join2 > test\_join1，也就是说**当已经有可迭代对象时，我们应该使用join方法，否则的话，构建可迭代对象再使用join方法，并不比直接使用+方法快**。
 
### 2.3 列表构造
 
 python有好几种循环构造列表的方法，for语句是用的最多的；map内置函数可以认为是将for代码移动到C层；而列表解析由于语法上的优势也很快；如果不需要生成列表的话，生成器表达式是最好的。
```python
oldlist = ['alpha', 'beta', 'gamma', 'delta']

def test_for():
    newlist = []
    upper = str.upper
    for word in oldlist:
        newlist.append(upper(word))

def test_map():
    upper = str.upper
    newlist = map(upper, oldlist)

def test_comprehension():
    upper = str.upper
    newlist = [upper(s) for s in oldlist]

def test_comprehension2():
    newlist = [s.upper() for s in oldlist]

def test_generator():
    upper = str.upper
    iterator = (upper(s) for s in oldlist)

if __name__ == '__main__':
	import timeit

    print "for:", timeit.timeit("test_for()", setup="from __main__ import test_for")
    print "map:", timeit.timeit("test_map()", setup="from __main__ import test_map")
    print "com:", timeit.timeit("test_comprehension()", setup="from __main__ import test_comprehension")
    print "com2:", timeit.timeit("test_comprehension2()", setup="from __main__ import test_comprehension2")
    print "gen:", timeit.timeit("test_generator()", setup="from __main__ import test_generator")
	
[out]:
for: 1.2300951343
map: 0.948517748575
com: 0.981668887338
com2: 0.692309199551
gen: 0.646842043714
```
从实验中我们可以看出，用生成器表达式生成可迭代对象是最快的，因为它只需要生成一个可迭代对象，而不需要生成列表；用for循环效率最差；那么map和列表解析应该用哪一个呢？从结果来看，当都用全局函数str.upper时，map会更快点；但我们用列表解析时可以用字符串自己的upper方法，这时它的效率又高于map不少，所以说，对于效率优化，并没有绝对的哪种方法优于另外一种，应该视不同的情况用不同的方法，同时，测试很关键，而不应该用我们印象中的“好”的方法。正常来说，构造列表我们应该使用列表解析或者map方法；Stack Overflow上有一篇帖子[Python List Comprehension Vs. Map](https://stackoverflow.com/questions/1247486/python-list-comprehension-vs-map)回答了在什么情况下应该使用什么方法，即：**当不使用lambda表达式时，map会稍微快一点；而当需要使用lambda表达式时，列表解析会更快**。下面的实验验证了这一点，我们仍然使用上面的例子，只是这次map和列表解析内的函数换成了lambda表达式。

```python
oldlist = ['alpha', 'beta', 'gamma', 'delta']

def test_map():
	f = lambda s: s.upper()
	newlist = map(f, oldlist)

def test_comprehension():
	f = lambda s: s.upper()
	newlist = [f(s) for s in oldlist]

if __name__ == '__main__':
	import timeit
	print "map:", timeit.timeit("test_map()", setup="from __main__ import test_map")
	print "com:", timeit.timeit("test_comprehension()", setup="from __main__ import test_comprehension")

[out]:
map: 1.24160480807
com: 1.10327335587
```
可以看到，当使用lambda时，列表解析会比map快点。

### 2.4 缓存类/实例方法

在python中，[bound & unbound method](https://www.jianshu.com/p/4b871019ef96#)都是临时的实例对象，即**每次调用bound和unbound方法都会创建一个新对象，并重构参数列表，并在调用完释放**。所以<font color=red>缓存bound & unbound method对象，可以提升性能</font>。
```python
oldlist = ["alpha", "beta", "gamma", "delta"]

def test_non_dot():
	upper = str.upper
	newlist = []
	append = newlist.append
	for word in oldlist:
		append(upper(word))

def test_dot():
	newlist = []
	for word in oldlist:
		newlist.append(str.upper(word))

if __name__ == '__main__':
	import timeit
	print "dot:", timeit.timeit("test_dot()", setup="from __main__ import test_dot")
	print "non_dot:", timeit.timeit("test_without_dot()", setup="from __main__ import test_without_dot")
	
[out]:
dot: 1.29089502165
non_dot: 1.09373510036
```
从实验结果可知，缓存了方法，可以节省15%左右的性能。

### 2.5 避免import语句开销
有时候为了加快模块的启动，特别是某个模块也许不会被用到时，我们会把Import语句写到函数里面——这是一种“延迟”的优化方法，即把工作延迟到真正需要的时候进行。但是要避免把这种写法写到要经常执行的函数中，这将非常耗性能：
```python
def doit1():
	import string
	string.lower('Python')

import string
def doit2():
    string.lower('Python')

if __name__ == '__main__':
	import timeit
	print "doit1:", timeit.timeit("doit1()", setup="from __main__ import doit1")
	print "doit2:", timeit.timeit("doit2()", setup="from __main__ import doit2")
	
[out]:
doit1: 0.813077810431
doit2: 0.320201488961
```
可以看到，在函数里进行了import操作，使得性能下降了154%；一种更好的惰性加载的方法是这样写的：
```python
string = None

def doit1():
	global string
	if string is None:
		import string
	string.lower('Python')

if __name__ == '__main__':
	import timeit
	print "doit1:", timeit.timeit("doit1()", setup="from __main__ import doit1")

[out]:
doit1: 0.333577688668
```
使用这种写法后，能够极大的节省每次检查是否已经导入模块的开销，性能和在模块层进行import类似，又可以利用lazy import的优点。

### 2.6 尽量减少函数调用次数
python的函数调用的开销相对来说是比较大的，特别和内建函数相比（用dir(\_\_builtins__)可以查看所有的内建函数，python 2.7.12共有145个内建函数），所以，我们应该尽量减少函数的调用次数：
```python
import time
x = 0
def sum1(i):
    global x
    x = x + i

list = range(100000)
t = time.time()
for i in list:
    sum1(i)

print "%.3f" % (time.time()-t)

[out]:
0.020
```
*vs.*
```python
import time
x = 0
def sum2(list):
    global x
    for i in list:
        x = x + i

list = range(100000)
t = time.time()
sum2(list)
print "%.3f" % (time.time()-t)

[out]:
0.008
```
同样是对1到100000进行累加，sum1调用了100000次，而sum2只调动了一次；suim2比sum1快了大约2.5倍，用C来写的话差距会更大。所以减少函数调用次数很重要，对于简单而调用次数很多的函数，我们应该尽量进行内联。

### 2.7 List, Tuple 和 Set

在python中，元组和列表具有相似的性能，但是元组用的内存会少一点（因为它们创建后就不可变） 
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-QlhqfyDb-1592356748179)(http://op1pzxins.bkt.clouddn.com/per_2.7_1.png)]  
这里使用了[python slots初探](http://blog.csdn.net/fitzzhang/article/details/78745469)这篇博客里介绍的[ipython_memeory_usage](https://github.com/ianozsvald/ipython_memory_usage)内存测量工具，可以看到，使用tuple大概节省了87%的内存，还是很可观的。  
**当进行迭代时**，使用tuple和list会比使用set快不少：
```python
from timeit import timeit

def iter_test(iterable):
	for i in iterable:
		pass


print 'set:', timeit("iter_test(iterable)", setup="from __main__ import iter_test; iterable = set(range(10000))", number=100000)

print 'list:', timeit("iter_test(iterable)", setup="from __main__ import iter_test; iterable = list(range(10000))", number=100000)

print 'tuple:', timeit("iter_test(iterable)", setup="from __main__ import iter_test; iterable = tuple(range(10000))",  number=100000)

[out]:
set: 15.6127514297
list: 11.2611329151
tuple: 11.0212029311
```
**当只用来检查一个值是不是应该存在时**，使用set会快很多，因为它底层是使用哈希表实现的。
```python

from timeit import timeit

def in_test(iterable):
	for i in range(1000):
		if i in iterable:
			pass

print 'set:', timeit("in_test(iterable)", setup="from __main__ import in_test; iterable = set(range(1000))", number=10000)
print 'list:', timeit("in_test(iterable)", setup="from __main__ import in_test; iterable = list(range(1000))", number=10000)
print 'tuple:', timeit("in_test(iterable)", setup="from __main__ import in_test; iterable = tuple(range(1000))", number=10000)

[out]:
set: 0.558294298165
list: 52.8850349101
tuple: 58.9864508751
```

**结论：当生成可迭代对象后并且不再进行改变，应该使用tuple节省内存；当生成集合用来进行检查某个值是否存在时，应该使用set来提高效率**。

## 3. 其它通用优化技巧

除了上面这些和python特定相关的优化技巧，还有一些和语言无关的优化技巧，以下列出几种，但仍然使用python来进行实验。

### 3.1 尽可能将if语句放在循环外面
这是在很多书本上看到的建议，然而这么做的原因，我并没有找到详尽的解释，我自己总结出来的原因有这么几个：
1. 如果可以把if放在循环外面，却放在循环里，就增加了很多不必要的判断
2. 在计算机体系结构层面，if放在循环里容易引起分支预测错误，而分支回退要耗很多指令周期
3. 在计算机体系结构层面，if放在循环里面会造成控制相关，影响指令并行(隐约记得在计算机体系结构这门课程中学过，然而记不太清了，有时间还是得复习复习相关知识=_=([计算机体系结构：量化研究方法](https://www.amazon.cn/dp/B00AX9IENS)，第三章：指令级并行及其开发))  

这是我自己的想法，欢迎大家指出不正确的地方，并进行补充。下面我们用一个实际的例子来证明这个建议的正确性。
假设我们有个循环，在循环中每隔n次迭代需要执行某种运算，因此我们可以这么写：
```python
def if_in_loop():
	sum = 0
	for i in xrange(256):
		if i % 64 == 0:
			sum += 2
		else:
			sum += 1
```
我们也可以把循环分成4个，从而避免使用if语句：
```python
def if_out_loop():
	sum = 0
	sum += 2
	for i in xrange(1, 64):
		sum += 1
	sum += 2
	for i in xrange(65, 128):
		sum += 1
	sum += 2
	for i in xrange(129, 192):
		sum += 1
	sum += 2
	for i in xrange(193, 256):
		sum += 1
```
它们的效率相差多少呢？我们实测下：
```python
print 'if_in_loop:', timeit("if_in_loop()", setup="from __main__ import if_in_loop", number=10000)
print 'if_out_loop:', timeit("if_out_loop()", setup="from __main__ import if_out_loop", number=10000)

[out]:
if_in_loop: 0.192863434315
if_out_loop: 0.0883827168327
```
消除循环里的if之后，效率提高了1.5倍。

### 3.2 使用大多数情况都能得到满足的if条件
现代计算机的处理器流水线一般都非常长（比如奔腾4流水线最多可包含20条指令），当出现未预测到的分支时，将刷新整个流水线，因此应避免错误预测发生。另外，处理器执行代码时，通常会预先执行它认为接下来将出现的代码，因此出现预测错误时，不但需要刷新流水线，还会处理永远不会执行的代码。为避免预测错误，应使用大多数情况都能得到满足的if条件。

## 4. 总结

本文首先介绍了python中进行性能测试的方法；然后介绍了几种python中通用的优化方法;最后介绍了几种和语言无关的优化方法。总结起来，本文给出的优化建议如下：
<font color=red>
1. 对python列表进行排序时，优先使用key参数；且相比内置的sorted方法，优先使用List内置的sort方法；
2. 对字符串进行拼接时，当已经有可迭代对象时，我们应该使用join方法，否则的话，构建可迭代对象再使用join方法，并不比直接使用+方法快；
3. 进行列表构建时，应该使用内建的map方法或者使用列表解析，具体使用哪一种应该视情况而定；当不使用lambda表达式时，map会稍微快一点；而当需要使用lambda表达式时，列表解析会更快；
4. 缓存类/实例的bound & unbound 方法，可以提升性能；
5. 应避免在高频函数中使用import语句的开销；
6. python函数调用的开销是比较大的，减少函数调用次数很重要，对于简单而调用次数很多的函数，我们应该尽量进行内联;
7. python中要生成可迭代对象时，当生成可迭代对象后并且不再进行改变，相比list,应该使用tuple节省内存；当生成集合用来进行检查某个值是否存在时，应该使用set来提高效率;
8. 不管使用哪种语言，都应该尽可能将if语句放在循环外面；
9. 应该使用大多数情况都能得到满足的if条件。

</font>

在写本文时，我并没有对python的实现进行深入研究，只是总结一些优化的经验以供自己在工作学习时参考，后续还会再进行完善补充；文中可能有很多不完善以及错误的地方，欢迎大家多多指正。路漫漫其修远兮，希望自己能够尽快抽出时间看python源码吧~
 
 
### 参考文献
 1. [PythonSpeed PerformanceTips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)(https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
 2. [计算机体系结构：量化研究方法](https://www.amazon.cn/dp/B00AX9IENS)，第三章：指令级并行及其开发
 3. [3D游戏编程大师技巧](https://item.jd.com/11047263.html),第16章：优化技术