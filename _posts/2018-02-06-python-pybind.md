---
layout: post
title:  "python调用C++之pybind11入门"
date:   2018-02-06 14:38:54
categories: python
tags: python
---

* content
{:toc}

python调用C/C\++有不少的方法，如boost.python, swig, ctypes, pybind11等，这些方法有繁有简，而pybind11的优点是对C++ 11支持很好，API比较简单，现在我们就简单记下Pybind11的入门操作。



## 1. pybind11简介与环境安装
pybind11是一个轻量级的只包含头文件的库，它主要是用来在**已有的** C\+\+代码的基础上做扩展，它的语法和目标非常像[Boost.Python](http://www.boost.org/doc/libs/1_58_0/libs/python/doc/)，但Boost.Python为了兼容现有的基本所有的C\++编译器而变得非常复杂和庞大，而因此付出的代价是很多晦涩的模板技巧以及很多不必要的对旧版编译器的支持。Pybind11摒弃了这些支持，它只支持python2.7以上以及C++ 11以上的编译器，使得它比Boost.Python更加简洁高效。
为了使用pybind11，我们需要支持C++ 11标准的编译器(GCC 4.8以上，VS 2015 Update 3以上)以及python 2.7以上的版本，还需要下载[CMake](https://cmake.org/cmake-tutorial/)，有了这些以后,
1. 首先，我们从 pybind11 github网址：https://github.com/pybind/pybind11 上下载源码。
2. cmake工程之前，要先安装pytest `pip install pytest`，否则会出错
3. 用CMake编译并运行测试用例：
```
mkdir build
cd build
cmake ..
cmake --build . --config Release --target check
```
如果所有测试用例都通过了，说明安装成功了。

## 2. python调用C++
下载编译好pybind11之后，我们就可以开始对着官方的[pybind11 Tutorial](http://pybind11.readthedocs.io/en/stable/index.html)进行学习了，详细的入门教程及语法请参考官方文档，这里，我们简单演示下如何编写供python调用的C\++模块.
首先，我们编写一个C++源文件，命名为<font color=red>`example.cpp`</font>
```C++
#include <pybind11/pybind11.h>
namespace py = pybind11;

int add(int i, int j)
{
	return i + j;
}

PYBIND11_MODULE(example, m)
{
	// optional module docstring
	m.doc() = "pybind11 example plugin";
	// expose add function, and add keyword arguments and default arguments
	m.def("add", &add, "A function which adds two numbers", py::arg("i")=1, py::arg("j")=2);
	
	// exporting variables
	m.attr("the_answer") = 42;
    py::object world = py::cast("World");
    m.attr("what") = world;
}
```
### 2.1 编译
然后，在windows下使用工具`vs2015 x86 Native Tools Command Prompt`  (因为我的python是32位版本，如果是64位版本的，请使用`vs2015 x64 Native Tools Command Prompt`)进行编译：  
`
cl example.cpp /I "H:/Allfiles/pybind11/include" /I "C:/Python27/include" /LD /Fe:example.pyd /link/LIBPATH:"C:/python27/libs/"
`  
编译成功后，会在`example.cpp`相同目录下生成`example.pyd`文件，这个就是python可以直接导入的库，运行：
```python
import example
example.add(3, 4)

[out]: 7
```
有了编译成功的模块，便可以使用我在另一篇博客[Python模块搜索路径](http://blog.csdn.net/fitzzhang/article/details/78988155)中提到的方法，将其用.pth或者PYTHONPATH的方法加入到python搜索路径，以后在我们自己的环境中就可以随时随地使用这个模块啦。
### 2.2 CMake的编译方法
当然，我们也可以使用CMake进行编译。首先写一个CMakeLists.txt
```
cmake_minimum_required(VERSION 2.8.12)
project(example)

add_subdirectory(pybind11)
pybind11_add_module(example example.cpp)
```
这里要求example.cpp放在和pybind11同一级的目录下，然后CMake，便会生成一个vs 2015的工程文件，用vs打开工程文件进行build，就可以生成`example.pyd`了。

## 3. C++调用python
使用pybind11，也很容易从C++里调用python脚本：  
首先，我们用vs 2015新建一个工程，并且将Python的包含目录和库目录，以及pybind11的包含目录配置到工程，我的配置如下：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-lZ3RbIen-1592357090303)(http://op1pzxins.bkt.clouddn.com/pybind11_1.png)]

然后，新建一个源文件`example.cpp`:
```C++
#include <pybind11/embed.h>  
#include <iostream>  

namespace py = pybind11;

int main() {
	py::scoped_interpreter python;

	//py::module sys = py::module::import("sys");
	//py::print(sys.attr("path"));

	py::module t = py::module::import("example");
	t.attr("add")(1, 2);
	return 0;
}
```
最后，在工程目录下加入脚本`example.py`:
```python
def add(i, j):
	print("hello, pybind11")
	return i+j
```
运行工程，得到如下的结果：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-dKE3dzti-1592357090305)(http://op1pzxins.bkt.clouddn.com/pybind11_2.png)]
运行成功！！！

### 总结
本文中我们简单介绍了怎么使用pybind11进行python和C++的相互调用，这些只是入门级别的例子，但是可以work了，这很重要。深入进行研究使用，还是等以后用到再说吧。

#### 参考文献
1. [pybind11 github](https://github.com/pybind/pybind11)
2. [pybind11 Tutorial](http://pybind11.readthedocs.io/en/stable/index.html)
3. [C++与Python的互操作](http://blog.csdn.net/thisisfangsheng/article/details/75610558)
4. [Building and testing a hybrid Python/C++ package](http://www.benjack.io/2017/06/12/python-cpp-tests.html)