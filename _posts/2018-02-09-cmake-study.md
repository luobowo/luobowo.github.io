---
layout: post
title:  "CMake入门学习笔记"
date:   2018-02-09 14:38:54
categories: c
tags: cmake
---

* content
{:toc}



## 一、CMake入门

### 1.1 CMake简介
CMake是一个跨平台的安装（编译）工具，可以用简单的语句来描述所有平台的安装(编译)过程。他能够输出各种各样的makefile或者project文件，能测试编译器所支持的C++特性,类似UNIX下的automake。cmake的流行其实要归功于KDE4的开发(似乎跟当年的svn一样，KDE将代码仓库从CVS迁移到SVN，同时证明了SVN管理大型项目的可用性)。

### 1.2 CMake初探——HelloWorld
接下来，我们用一个简单的例子来看看CMake的完整构建过程。我们的实验环境是Windows+VS 2015 update3 + CMake 3.10.0。
首先，我们建立一个新的文件夹`CMake/t1`，然后在下面建立main.c文件和CMakeLists.txt文件，内容如下：  
<font color=green>main.c</font>  
```C
#include<stdio.h>

int main()
{
	printf("Hello World from t1 Main!\n");
	return 0;
}
```
<font color=green>CMakeLists.txt</font>  
```
project(hello)
set(SRC_LIST main.c)
message(STATUS "This is binary dir " ${hello_BINARY_DIR})
message(STATUS "This is source dir " ${hello_SOURCE_DIR})
add_executable(hello ${SRC_LIST})
```
然后，我们在这个目录下执行`cmake .`(把cmake的执行目录加入环境变量即可执行),输出大概是这个样子：
```
-- Building for: Visual Studio 14 2015
-- The C compiler identification is MSVC 19.0.24215.1
-- The CXX compiler identification is MSVC 19.0.24215.1
-- Check for working C compiler: C:/Program Files (x86)/Microsoft Visual Studio
14.0/VC/bin/cl.exe
-- Check for working C compiler: C:/Program Files (x86)/Microsoft Visual Studio
14.0/VC/bin/cl.exe -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working CXX compiler: C:/Program Files (x86)/Microsoft Visual Studi
o 14.0/VC/bin/cl.exe
-- Check for working CXX compiler: C:/Program Files (x86)/Microsoft Visual Studi
o 14.0/VC/bin/cl.exe -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- This is binary dir H:/AllFiles/CMake/t1
-- This is source dir H:/AllFiles/CMake/t1
-- Configuring done
-- Generating done
-- Build files have been written to: H:/AllFiles/CMake/t1
```

你会发现，在目录下系统自动生成了CMakeFiles,CMakeCache.txt,cmake_install.cmake等文件，最关键的是，它自动生成了hello.sln工程文件，打开工程文件并编译即可得到可执行文件。

### 1.3 CMake指令简单解释
1. **`project(projectname [CXX] [C] [Java])`**  
这个指令用来定义工程名称，并可指定工程支持的语言，默认支持所有语言。同时这个指令隐式的定义了两个cmake变量:<projectname>\_BINARY\_DIR以及<projectname>\_SOURCE\_DIR，这两个变量会因是内部编译还是外部编译而指代不同的内容。同时cmake系统也帮助我们定义了**PROJECT_BINARY_DIR**和**PROJECT_SOURCE_DIR**，他们的值和上面两个变量的值一致，好处是不会随着工程名称的改变而改变。PROJECT_BINARY_DIR 指的是编译发生的当前目录，如果是内部编译，就相当于PROJECT_SOURCE_DIR也就是工程代码所在目录，如果是外部编译，指的是外部编译所在目录。
2. **`set(VAR [value] [CACHE TYPE DOCSTRING [FORCE]])`**  
set用来显示定义变量。
3. **`message([SEND_ERROR | STATUS | FATAL_ERROR] "message to display" ...)`**
这个指令用户向终端输出用户自定义信息，包含了三种类型：  
SEND_ERROR，产生错误，生成过程被跳过。  
SATUS，输出前缀为--的信息。  
FATAL_ERROR，立即终止所有 cmake 过程。  
4. **`add_executable(name ${SRC_LIST})`**  
定义了这个工程文件会生成一个文件名为name的可执行文件，相关的源文件是SRC_LIST中定义的源文件列表。

### 1.4 基本语法规则
1. 变量使用${}方式取值，但是在IF控制语句中是直接使用变量名；
2. 指令(参数 1 参数 2...),参数之间使用空格或分号分开,例如add_executable(hello  main.c func.c)或者add_executable(hello main.c;func.c)
3. 指令是大小写无关的，参数和变量是大小写相关的。这里需要特别解释的是作为工程名的hello和生成的可执行文件hello是没有任何关系的。

## 二、逐渐深入——一个更复杂的例子
to be continue...(最近较忙，等用到再慢慢学习)

#### 参考文献
1. [CMake Pratice](https://github.com/Akagi201/learning-cmake/blob/master/docs/cmake-practice.pdf)
2. [CMake Tutorial](https://cmake.org/cmake-tutorial/)
3. [CMake入门教程](http://blog.csdn.net/fan_hai_ping/article/details/42524205)