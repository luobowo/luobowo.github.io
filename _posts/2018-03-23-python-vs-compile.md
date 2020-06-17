---
layout: post
title:  "VS2015编译python 3.6.4源码"
date:   2018-03-23 14:38:54
categories: python
tags: python
---

* content
{:toc}

用了很久python, 最近决定在windows下编译python的源代码，还是遇到了几个坑，花了几个小时==谨记此文，希望为后来者避开这些坑。




1. 首先，我们从官网下载[python 3.6.4的源代码](https://www.python.org/downloads/release/python-364/)，选择Gzipped source tarball 或者 XZ compressed source tarball
2. 然后，我们解压开源码，进入到PCbuild目录，里面有VS的工程文件`pcbuild.sh`，**仔细阅读下readme.txt**，最重要的信息是需要的VS的版本是VS2015，编译一个可以用的CPython解析器最少需要编译python和pythoncore
3. 我们就只编译python和pythoncore，需要别的功能，请参照readme.txt的内容，用VS2015打开`pcbuild.sh`，设置解决方案的配置属性，只保留python和pythoncore，如下图所示：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-UeARMrn9-1592357324167)(http://op1pzxins.bkt.clouddn.com/vs2015cpython_1.png)]
4. 编译解决方案，这时有可能会出错：`windows sdk version 10.0.15063 was not found`，解决方案是根据[Fix python 3.6 build failure with VS 2015 and WinSDK!=10.0.15063](https://github.com/isuruf/cpython/commit/9432a2c7f63b3bb55e8066e91eade81321154476)所说的方法，打开`python.props`, 将第77行
```
<DefaultWindowsSDKVersion>10.0.15063.0</DefaultWindowsSDKVersion>
```
更改为 
```
<DefaultWindowsSDKVersion Condition="$(_RegistryVersion) == '10.0.15063'">10.0.15063.0</DefaultWindowsSDKVersion>
```
5.然后就能编译成功啦，在`PCbuild`同目录层次下会生成`Lib`文件夹，在`PCbuild`目录下会生成`win32`文件夹，里面根据你编译的是debug还是release版本，会生成`python_d.exe`或者`python.exe`，打开它，就能运行了。

6.如果不想安装VS2010并且想编译Python 2.7的话，可以参考[使用 Visual Studio 2015编译 Python 2.7.11](http://disenone.github.io/2016/06/01/vs2015-python)这篇文章，如果编译的是python 2.7.14的话，按照上面的文章设置完后，还会出现错误，可以参考 https://bugs.python.org/issue25759 的方法。