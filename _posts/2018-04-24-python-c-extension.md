---
layout: post
title:  "浅谈Python C扩展"
date:   2018-04-24 00:13:01
categories: python c
tags: python c
---

* content
{:toc}


很多时候，我们需要写Python的C扩展，例如为了提高速度，用一些C的库等等。本文首先整理了python调用C扩展以及在C中调用python的方法；然后重点分析了CPython API中的引用计数问题。




在python应用中，为了对性能进行优化，我们常常需要写python的C扩展，将一些关键代码用C进行重写以提高性能；同时，我们也可以用在C中调用python的方法，例如写回调函数等。不管是python调用C，还是C调用python，最重要的是引用计数的管理，这也是最容易引起问题的地方。本文首先从简单的范例开始讲解python和C的互相调用，然后重点学习CPython API的引用计数问题。对python C扩展比较熟的可以直接跳过前面两部分，只看第三部分（大神请忽视本文）。

## 1. Python C 扩展基础

### 1.1 主要步骤
首先，我们看看用C写一个python扩展需要哪些步骤： 

* 包含头文件*Python.h*
* 你需要作为python接口的C函数
* 一个将你的函数映射为python接口的映射表
* 一个初始化函数

#### 1.1.1 Python.h头文件

这个头文件包含了所有的用来将你的模块hook到python解析器的CPython API，而且**你必须将这个头文件写在任何标准头文件之前**，这是因为这个头文件可能定义了一些影响标准头文件的预处理宏。

#### 1.1.2 C函数
python C 扩展的函数定义一般是下面的三种形式之一：
```C
static PyObject *MyFunction( PyObject *self, PyObject *args );

static PyObject *MyFunctionWithKeywords(PyObject *self,  PyObject *args, PyObject *kw);

static PyObject *MyFunctionWithNoArgs( PyObject *self );
```
Python中的函数都返回PyObject类型的指针，没有像C那种返回void类型的；如果你的函数不想返回一个值的话，Python定义了一个宏`Py_RETURN_NONE`，它等价于在脚本层返回None。
你的C函数应该是个静态函数，名字是任意的，但一般命名为`模块名_函数名`的形式，所以，一个典型的函数长这样：
```C
static PyObject *modulename_func(PyObject *self, PyObject *args) {
   /* Do something here. */
   Py_RETURN_NONE;
}
```

#### 1.1.3 方法映射表

方法映射表就是PyMethodDef结构的数组，而PyMethodDef结构体长这样：
```C
struct PyMethodDef {
   char *ml_name;
   PyCFunction ml_meth;
   int ml_flags;
   char *ml_doc;
};
```
其各个参数的意义如下：

* **ml_name**: 这是暴露给python程序的函数名；
* **ml_meth**: 这是指向1.1.2所讲的函数的指针，也就是真正函数定义的地方；
* **ml_flags**: 这告诉python解析器想用三种函数签名的哪一种，一般来说，它的值是METH\_VARARGS；如果你想传入关键字参数的话，也可以与MET_KEYWORDS进行或运算；当然，如果你不想接受任何参数的话，可以给其赋值为METH\_NOARGS；
* **ml_doc**: 这是函数的文档字符串，如果你不想写的话，直接给其赋值为NULL。

最后要注意的是，这个映射表应该以一个由NULL和0组成的结构体进行结尾。所以，一个方法映射表应该长这样：
```C
static PyMethodDef module_methods[] = {
   { "func", (PyCFunction)module_func, METH_VARARGS, NULL },
   { NULL, NULL, 0, NULL }
};
```

#### 1.1.4 初始化函数

你的扩展模块的最后一部分就是初始化函数了，它会在模块被导入时被python解析器调用。**初始化函数必须被命名为<font color=red>initModuleName</font>**，这里ModuleName表示你的模块名。  
这个初始化函数需要从你构建的库中导出，所以Python头文件里定义了PyMODINIT_FUNC来进行这项工作，你需要做的就是在定义函数时使用它；这个函数也应该是**你的模块中唯一一个非static的项**。这个初始化函数的原型一般是这样的：
```C
PyMODINIT_FUNC initModuleName() {
   Py_InitModule3(ModuleName, module_methods, "docstring...");
}
```
py_InitModule3的参数定义如下：

* **module_name**: 被导出的模块名；
* **module_methods**: 上面所定义的映射表；
* **docstring**: 你想要给你的模块的注释；

---
将上面的所有步骤结合在一起，一个C扩展模块看起来长这样：
```C
#include <Python.h>

static PyObject *module_func(PyObject *self, PyObject *args) {
   /* Do your stuff here. */
   Py_RETURN_NONE;
}

static PyMethodDef module_methods[] = {
   { "func", (PyCFunction)module_func, METH_VARARGS, NULL },
   { NULL, NULL, 0, NULL }
};

PyMODINIT_FUNC initModule() {
   Py_InitModule3(Module, module_methods, "docstring...");
}
```
---

### 1.2 Example

在1.1节我们已经覆盖了一个简单C扩展模块所需的所有知识点，现在我们通过一个实例来实践下；我们的C模块实现的功能是两个浮点数的乘法和除法，最后编译成名为`example`的模块。  
首先，根据上面的知识点，我们写一个example.c源文件，内容如下：
```C
#include <Python.h>

static PyObject* example_mul(PyObject* self, PyObject*args)
{
	float a, b;
	if(!PyArg_ParseTuple(args, "ff", &a, &b))
	{
		return NULL;
	}
	return Py_BuildValue("f", a*b);
}

static PyObject* example_div(PyObject* self, PyObject*args)
{
	float a, b;
	if(!PyArg_ParseTuple(args, "ff", &a, &b))
	{
		return NULL;
	}
	return Py_BuildValue("f", a/b);  // to deal with b == 0
}

static char mul_docs[] = "mul(a, b): return a*b\n";
static char div_docs[] = "div(a, b): return a/b\n";

static PyMethodDef example_methods[] =
{
	{"mul", (PyCFunction)example_mul, METH_VARARGS, mul_docs},
	{"div", (PyCFunction)example_div, METH_VARARGS, div_docs},
    {NULL, NULL, 0, NULL}
};

void PyMODINIT_FUNC initexample(void)
{
	Py_InitModule3("example", example_methods, "Extension module example!");
}
```
这里PyArg\_ParseTuple和Py\_BuildValue分别用来解析python的参数和构建python的值，这两个函数将在下面讲到，这里需要注意的是因为我们要导出`example`这个模块，所以最后的**initModuleName**的ModuleName以及调用的Py_InitModule3的第一个参数的名字都是example.

#### 1.2.1 编译和安装扩展

有了这个源文件，我们应该怎么编译和安装这个扩展，使得它成为我们可以导入的python模块的一部分呢？答案是[distutils](https://docs.python.org/2/distutils/index.html#distutils-index)模块，它就是用来发布python模块的(官方推荐使用[setuptools](https://setuptools.readthedocs.io/en/latest/)，但我没有去研究怎么用).  
我们首先定义个setup.py脚本文件，内容如下：

```python
from distutils.core import setup, Extension
setup(name="exampleAPP", version="1.0", ext_modules=[Extension("example", ["example.c"])])
```
这里需要注意的是ext_modules里的Extension的模块名必须和我们想要导出的模块名相同（这里就是exmaple)，否则会出现`LINK : error LNK2001: unresolved external symbol`的错误，然后我们用下面这个命令进行编译与安装：
```python
python setup.py install
```
安装成功后，就会在python_path/Lib/site-packages下面生成example.pyd这个模块和exampleAPP-1.0-py2.7.egg-info这个文件，就可以导入和使用了：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-Qve6eYnn-1592357484215)(http://op1pzxins.bkt.clouddn.com/pcallc_1_2.png)]

**注意**：在windows下，使用vs进行编译的的话，可能会出错：<font color=red>**error: Unable to find vcvarsall.bat**</font>  
在StackOverflow上找到了答案：[error: Unable to find vcvarsall.bat](https://stackoverflow.com/questions/2817869/error-unable-to-find-vcvarsall-bat)，原因是当用setup.py去安装包时，python 2.7会寻找 Visual Studio 2008(python 2.7就是用VS2008编译的)，找不到的话就会报这个错；一种trick的方法是根据你安装的VS版本，在执行setup.py之前先执行以下命令：
```
Visual Studio 2010 (VS10): SET VS90COMNTOOLS=%VS100COMNTOOLS%  
Visual Studio 2012 (VS11): SET VS90COMNTOOLS=%VS110COMNTOOLS%
Visual Studio 2013 (VS12): SET VS90COMNTOOLS=%VS120COMNTOOLS%
Visual Studio 2015 (VS14): SET VS90COMNTOOLS=%VS140COMNTOOLS%
```
但这种做法并不保险，而且用与编译python本身不同版本的编译器去编译python C扩展还可能引起不兼容问题，正确的做法是下载Visual C++ 2008或者 [Microsoft Visuial C++ Compiler for Python](https://www.microsoft.com/en-us/download/details.aspx?id=44266)(需要setuptools和wheel这两个python包，而且必须要用setuptools.setup()而不是distutils来进行安装。)

### 1.3 参数提取——PyArg\_ParseTuple函数

上面的例子中，脚本层传入的参数会存在PyObject* args所指向的PyObject里面，那么我们怎么提取出参数呢？答案是使用PyArg_ParseTuple函数，它的原型是这样的：
```C
int PyArg_ParseTuple(PyObject* tuple,char* format,...)
```
这个函数遇到错误返回0，返回别的数字代表正确。tuple就是C函数传进来的第二个参数，format是描述参数格式的字符串，里面的格式码意义如下：  
Code | C type | Meaning
 --- | --- | ---
  c  | char   | A Python string of length 1 becomes a C char
  d  | double | A Python float becomes a C double
  f  | float  | A Python float becomes a C float
  i  | int    | A Python int becomes a C int
  l	 | long	  | A Python int becomes a C long.
  L	 | long long | A Python int becomes a C long long
  O	 | PyObject* | Gets non-NULL borrowed reference to Python argument.
  s	 | char\*    | Python string without embedded nulls to C char*.
  s# | char*+int | Any Python string to C address and length.
  t# | char*+int | Read-only single-segment buffer to C address and length.
  u	 | Py_UNICODE* | Python Unicode without embedded nulls to C.
  u# | Py_UNICODE*+int | Any Python Unicode C address and length.
  w# | char*+int | Read/write single-segment buffer to C address and length.
  z	 | char\* | Like s, also accepts None (sets C char* to NULL).
  z# | char\*+int | Like s#, also accepts None (sets C char* to NULL).
(...) |	as per ... | A Python sequence is treated as one argument per item.
  \| |           | The following arguments are optional.
  :	 |	         | Format end, followed by function name for error messages.
  ;	 |       	 | Format end, followed by entire error message text.

剩余的参数就是**变量的地址**，而变量的类型由格式串的格式码决定。要解析带有关键字的参数的话，请使用PyArg\_ParseTupleAndKeywords
```C
int PyArg_ParseTupleAndKeywords(PyObject *args, PyObject *kw, const char *format, char *keywords[], ...)
```

### 1.4 返回值和Py_BuildValue

Python C 函数的返回值都是PyObject\*类型的（错误返回NULL），如果不想返回任何值，就是用宏`Py_RETURN_NONE`。Py\_BuildValue刚好和PyArg\_ParseTuple相反，它是用来将C的变量构建为Python的PyObject\*的(但这时传入的不是地址，而是值)，它的原型如下：
```C
PyObject* Py_BuildValue(char* format,...)
```
这个字符串格式码和上面的类似，下面列出了常用的字节码：
 Code | C type | Meaning
 ---  | ---    | ---
  c	  | char   | A C char becomes a Python string of length 1.
  d	  | double | A C double becomes a Python float.
  f   | float  | A C float becomes a Python float.
  i	  | int	   | A C int becomes a Python int.
  l	  | long   | A C long becomes a Python int.
  N	  | PyObject* | Passes a Python object and steals a reference.
  O	  | PyObject* | Passes a Python object and INCREFs it as normal.
  O&  | convert+void* | Arbitrary conversion
  s	  | char\*  | C 0-terminated char* to Python string, or NULL to None.
  s#  | char\*+int | C char* and length to Python string, or NULL to None.
  u	  | Py_UNICODE* | C-wide, null-terminated string to Python Unicode, or NULL to None.
  u#  | Py_UNICODE*+int | C-wide string and length to Python Unicode, or NULL to None.
  w#  | char*+int | Read/write single-segment buffer to C address and length.
  z  | char\* | Like s, also accepts None (sets C char* to NULL).
 z#	 | char\*+int | Like s#, also accepts None (sets C char* to NULL).
(...) | as per ... | Builds Python tuple from C values.
[...] |	as per ... | Builds Python list from C values.
{...} | as per ... | Builds Python dictionary from C values, alternating keys and values.

{...} 用来从偶数个key和value隔开的C的值中构建字典，例如`Py_BuildValue("{issi}", 23, "zig", "zag", 42)`返回一个python的字典：{23:'zig', 'zag':42}.

### 1.5 错误和异常处理

当一个函数失败时，Python解释器的一个重要约定是返回一个错误值（一般是NULL）并设置3个全局静态变量，分别对应Python的sys.exec\_type, sys.exec\_value和sys.exec\_traceback. 最先检测到异常的函数应该报告并设置全局变量，其它调用它的函数应该只是返回异常值，例如：当**f**调用**g**并检测到**g**失败了，它应该返回一个错误值(一般是NULL或-1),它不应该调用任何一个PyErr\_\*()函数，这应该是g调用的。**f**的调用者也应该返回一个错误值，以此类推。   
python API定义了一些函数来设置并检查各种异常：  
(1)PyErr\_SetString(PyObject\* type, const char\* message):  
type一般是一个预定义的对象，例如PyExc_ZeroDivisionError，C字符串用来说明异常出现的原因   
(2)PyErr_SetObject(PyObject\* type, PyObject\* value):  
最常用  
(3)PyErr_Occurred():  
用来检查是否设置了一个异常  
(4)如果想要忽视一个异常而不传递给解析器的话，可以调用PyErr_Clear()函数  
(5)所有直接调用malloc()或者realloc()的函数失败的话，必须要调用PyErr_NoMemory()，并且返回失败标志  

### 1.6 小结

本节讲解了写一个C模块的一些基本知识点和约定的异常处理流程，并用一个实例展示了如何编译与调用C模块，下一节我们讲下如何从C中调用python的方法。

## 2. C调用Python

C调用Python的方法也很简单，下面我们以windows+VS2015+python2.7讲解下如何用C调用Python.
首先，我们新建一个工程，并将python的包含目录和库目录设置到工程的目录里面去（注意，这里要设置release版本的，因为我们下载的python是release版本的，如果用debug的话，会在编译时出现Error: cannot open file 'python27_d.lib'错误），如下图所示：

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-PA37xhNr-1592357484218)(http://op1pzxins.bkt.clouddn.com/pc_2_1.png)]

然后，我们新建源文件，内容如下所示：

```C
#include <Python.h>

int main(int argc, char *argv[])
{
	PyObject *pName, *pModule, *pDict, *pFunc;
	PyObject *pArgs, *pValue;
	int i;

	if (argc < 3) {
		fprintf(stderr, "Usage: call pythonfile funcname [args]\n");
		return 1;
	}

	Py_Initialize();        // Initialize the Python Interpreter
	pName = PyString_FromString(argv[1]);    // Build the name object

	pModule = PyImport_Import(pName);
	Py_DECREF(pName);

	if (pModule != NULL) {
		pFunc = PyObject_GetAttrString(pModule, argv[2]);
		/* pFunc is a new reference */

		if (pFunc && PyCallable_Check(pFunc)) {
			pArgs = PyTuple_New(argc - 3);
			for (i = 0; i < argc - 3; ++i) {
				pValue = PyInt_FromLong(atoi(argv[i + 3]));
				if (!pValue) {
					Py_DECREF(pArgs);
					Py_DECREF(pModule);
					fprintf(stderr, "Cannot convert argument\n");
					return 1;
				}
				/* pValue reference stolen here: */
				PyTuple_SetItem(pArgs, i, pValue);
			}
			pValue = PyObject_CallObject(pFunc, pArgs);
			Py_DECREF(pArgs);
			if (pValue != NULL) {
				printf("Result of call: %ld\n", PyInt_AsLong(pValue));
				Py_DECREF(pValue);
			}
			else {
				Py_DECREF(pFunc);
				Py_DECREF(pModule);
				PyErr_Print();
				fprintf(stderr, "Call failed\n");
				return 1;
			}
		}
		else {
			if (PyErr_Occurred())
				PyErr_Print();
			fprintf(stderr, "Cannot find function \"%s\"\n", argv[2]);
		}
		Py_XDECREF(pFunc);
		Py_DECREF(pModule);
	}
	else {
		PyErr_Print();
		fprintf(stderr, "Failed to load \"%s\"\n", argv[1]);
		return 1;
	}
	Py_Finalize();     // Finish the Python Interpreter
	return 0;
}
```
我们在工程目录下新建`Mul.py`，内容如下：
```python
def multiply(a,b):
    print "Will compute", a, "times", b
    c = 0
    for i in range(0, a):
        c = c + b
    return c
```
运行，得到结果：
```
Will compute 3 times 4
Result of call: 12
请按任意键继续. . .
```
C调用Python的源代码还是很直观的，其中最难的部分是那些Py\_DECREF()和Py\_XDECREF()，这是什么？第一次看确实会一头雾水，别急，下面一节我们就要讲python C API的引用计数。

## 3. Reference Counts

在使用Python C API时，最容易出错的地方就是引用计数的管理。不管是内存泄露还是非法内存放访问，对于程序来说都是致命的，下面我们就简单讲讲CPython API中的引用计数。

### 3.1 CPython引用计数简介

在C/C++中，程序员负责动态内存的申请与释放释放，在C中，这是通过调用`malloc()`/`free()`来实现的；如果只进行了内存申请而没有手动释放就会造成内存泄露，而如果使用已释放的内存就会造成非法内存访问(use freed memory)；由于CPython大量使用`malloc()`和`free()`,所以需要一种策略来避免内存泄露和非法内存访问，CPython是通过使用引用计数(*reference counting*)来实现的。    
CPython具有两个宏`Py_INCREF(x)`和`Py_DECREF(x)`(`Py_XINCREF`和`Py_XDECREF`的作用和它们类似，只是会检查传进去的指针是否为空)，分别用来增加和减少引用计数，此外，`Py_DECREF`也会在引用计数减少到0后释放对象；那么问题来了，什么时候使用`Py_INCREF(x)`和`Py_DECREF(x)`呢？  
要回答前面的这个问题，我们要首先引入CPython的一些术语。**在CPython中，没有人拥有一个对象，拥有的是对象的引用**；引用的拥有者负责在引用不再引用这个对象时对它调用`Py_DECREF`,引用的拥有权也可以转移。在CPython中，使用术语"New","Stolen"和"Borrowed" references来表示三种引用，这些术语其实是表明谁是引用的真正拥有者，即谁负责对引用进行处理。

* **New References:**  
当新建一个`PyObject`对象时，就产了一个New Reference，例如当调用`PyInt_FromLong`时。New Reference意味着你拥有这个引用。
* **Stolen References:**  
这一般出现在函数调用时将一个引用传进去当参数时，这个函数会假设现在它拥有这个引用，即它会“偷取”这个引用，这意味着当你调用这个函数后，你就不再拥有这个参数的引用。例如当调用`PyList_SetItem(PyObject* list, index, PyObject* item)`后，你就不再拥有对item的引用
* **Borrowed References:**  
Borrowed Reference一般出现在查看一个`PyObject`时，例如从一个列表里面获取一个成员。借来的引用不应该调用`Py_DECREF`，而且它持有对象的时间不应该比引用的拥有者长，如果在引用的拥有者已经释放这个引用后，还是访问借来的引用，就会造成非法内存访问；借来的引用也可以通过调用`Py_INCREF`变为拥有的引用。  

### 3.2 CPython 引用拥有权规则

#### 3.2.1 拥有权规则简单概括

在3.1节我们简单介绍了Cpython的引用计数，现在我们概括下引用的拥有权的规则，主要分为调用函数时作为参数传入的引用拥有权转移规则和作为函数返回值的引用的拥有权的转移规则:

* **作为函数返回值时:**  
(1)大部分返回引用的函数都会将这个引用的拥有权转移到函数调用者(即返回新的引用)，例如`PyInt_FromLong`和`Py_BuildValue`等;  
(2)然而也有少数例外，例如<font color=red>`PyTuple_GetItem()`,`PyList_GetItem()`,`PyDIct_GetItem()`和`PyDict_GetItemString()`</font>，它们返回的是**borrowed Reference**。`PyImport_AddModule()`返回的也是借来的引用。
* **作为参数传递时:**  
(1)在你将一个对象的引用传递进另一个函数时，一般来说这个函数会从你借这个引用，也就是说，在函数，一般参数的引用是borrowed reference;  
(2)有两个比较重要的例外，<font color=red>**`PyTuple_SetItem()`和`PyList_SetItem()`</font>，它们会从你这偷取引用(steal reference)**，这意味着当你把引用传递给这些函数时，这些函数就会拥有这些引用，而你不再拥有这些引用。  

#### 3.2.2 引用拥有权例外总结

就像上节总结的一样，我们只要记住一般来说，作为返回值的引用是一个新的引用，我们要负责其释放；而作为参数传入的引用一般是borrowed reference，我们用完就可以了；而那些例外的函数总结如下：

* 从参数中steal reference的：  
```python
PyCell_SET (but not PyCell_Set)
PyList_SetItem
PyList_SET_ITEM
PyModule_AddObject
PyTuple_SetItem, PyTuple_SET_ITEM
```
* 返回borrowed reference的函数
```python
all PyArg_Xxx functions
PyCell_GET (but not PyCell_Get)
PyDict_GetItem
PyDict_GetItemString
PyDict_Next
PyErr_Occurred
PyEval_GetBuiltins
PyEval_GetFrame
PyEval_GetGlobals
PyEval_GetLocals
PyFile_Name
PyFunction_GetClosure
PyFunction_GetCode
PyFunction_GetDefaults
PyFunction_GetGlobals
PyFunction_GetModule
PyImport_AddModule
PyImport_GetModuleDict
PyList_GetItem, PyList_GETITEM
PyMethod_Class, PyMethod_GET_CLASS
PyMethod_Function, PyMethod_GET_FUNCTION
PyMethod_Self, PyMethod_GET_SELF
PyModule_GetDict
PyObject_Init
PyObject_InitVar
PySequence_Fast_GET_ITEM
PySys_GetObject
PyThreadState_GetDict
PyTuple_GetItem, PyTuple_GET_ITEM
PyWeakref_GetObject, PyWeakref_GET_OBJECT
Py_InitModule
Py_InitModule3
Py_InitModule4
```

### 3.3 关于引用的易错点

上面两节我们介绍了引用以及引用的拥有权规则，现在我们讲讲CPython中引用中容易犯的错误，引用主要容易出两类错误:  
(1)引用不再指向对象后没有减少引用计数导致内存泄露，类似于在C中调用了`malloc()`而没有调用`free()`，例如： 
```C
static PyObject *bad_incref(PyObject *pObj) {
    Py_INCREF(pObj);
    /* ... a metric ton of code here ... */
    if (error) {
        /* No matching Py_DECREF, pObj is leaked. */
        return NULL;
    }
    /* ... more code here ... */
    Py_DECREF(pObj);
    Py_RETURN_NONE;
}
```
(2)在对象释放后仍然通过引用去访问对象，类似于在C中`free()`以后去获取对象或者使用野指针([dangling pointer](https://en.wikipedia.org/wiki/Dangling_pointer))，例如：  
```C
static PyObject *bad_incref(PyObject *pObj) {
    /* Forgotten Py_INCREF(pObj); here... */

    /* Use pObj... */

    Py_DECREF(pObj); /* Might make reference count zero. */
    Py_RETURN_NONE;  /* On return caller might find their object free'd. */
}
```
函数返回后，调用者可能会使用已经释放掉的pObj，这是一个典型的access-after-free错误。  
上面举例所示的错误都是小心点就可以避免的，然而有些引用错误就比较隐蔽，也是我们需要特别注意的地方，下面我们通过举例来进行说明。

#### 3.3.1 New References比较容易出现的错误
对于New Reference，我们最容易犯的错误就是将一个函数返回的New Reference作为临时变量传进函数的参数，由于大部分函数的参数传递都是以Borrowed Reference进行的，就会导致这个New Reference没有人对其进行引用计数管理，从而导致内存泄露。以下的函数是将两个数进行详见，我们用第一节的方法将其编译成python的扩展模块，并将example\_substract导出为sub接口进行调用。
```C
static PyObject* subtract_long(long a, long b) {
    PyObject *pA, *pB, *r;

    pA = PyLong_FromLong(a);        /* pA: New reference. */
    pB = PyLong_FromLong(b);        /* pB: New reference. */
    r = PyNumber_Subtract(pA, pB);  /*  r: New reference. */
    Py_DECREF(pA);                  /* My responsibility to decref. */
    Py_DECREF(pB);                  /* My responsibility to decref. */
    return r;                       /* Callers responsibility to decref. */
}

static PyObject* example_subtract(PyObject* self, PyObject* args)
{
	PyObject* result;
	long a, b;
	if(!PyArg_ParseTuple(args, "ll", &a, &b))
	{
		return NULL;
	}
	result = subtract_long(a, b);
	return result;
}
```
然而，一个很容易犯的错误就是在调用`PyNumber_Subtrace`时，我们直接将`PyLong_FromLong(x)`传进去，由于`PyNumber_Substract()`只会借取引用，它并不会释放引用，这时返回的New Reference并没有对其进行`Py_DECREF`，就会导致内存泄露，如下example\_bad\_subtrace，我们将其导出为bad\_sub接口：
```C
static PyObject* bad_subtract_long(long a, long b) {
    PyObject *r;
    r = PyNumber_Subtract(PyLong_FromLong(a), PyLong_FromLong(b));  /*  r: New reference. */
    return r;                       /* Callers responsibility to decref. */
}

static PyObject* example_bad_subtract(PyObject* self, PyObject* args)
{
	PyObject* result;
	long a, b;
	if(!PyArg_ParseTuple(args, "ll", &a, &b))
	{
		return NULL;
	}
	result = bad_subtract_long(a, b);
	return result;
}
```
用[ipython_memory_usage](https://github.com/ianozsvald/ipython_memory_usage)对内存进行测量，分别调用`example.sub`和`example.bad_sub`，看是否有内存泄露：
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-3QntNl0T-1592357484219)(http://op1pzxins.bkt.clouddn.com/pcc_memory_leak1.png)]

从结果可以看到，每调用100000次左右的`example.bad_sub`，就会导致3M左右的内存泄露，从而印证了我们的猜想。

### 3.3.2 Stolen References比较容易出现的错误

CPython中Stolen Reference的情况不多，两个最重要的需要记住的就是`PyList_SetItem`和`PyTuple_SetItem`，对于Stolen Reference，我们只需要记住当引用传进这两个函数后，我们便不再拥有对引用的拥有权，也就不能再对其进行`Py_DECREF`了。
```C
static PyObject *make_tuple(void) {
    PyObject *r;
    PyObject *v;

    r = PyTuple_New(3);         /* New reference. */
    v = PyLong_FromLong(1L);    /* New reference. */
    /* PyTuple_SetItem "steals" the new reference v. */
    PyTuple_SetItem(r, 0, v);
    /* This is fine. */
    v = PyLong_FromLong(2L);
    PyTuple_SetItem(r, 1, v);
    Py_DECREF(v);    /* Now we are interfering with r's internals. */
    /* More common pattern. */
    PyTuple_SetItem(r, 2, PyUnicode_FromString("three"));
    return r; /* Callers responsibility to decref. */
}
```
当v被传递给`PyTuple_SetItem`后，v的引用被偷走了，它成为了一个borrowed reference, 再对它调用`Py_DECREF`可能会引起未知的行为。

### 3.3.3 Borrowed References比较容易出现的错误

在引用出现错误的地方，最奇怪的bug常常和borrowed reference有关。
例如我们用borrowed reference来操作列表的最后一个元素，操作步骤如下：
* 从列表中得到最后一个元素的borrowed reference
* 对列表进行操作`do_something()`
* 操作最后一个元素的borrowed reference，这里只是简单的打印它。  
代码如下：
```C
static PyObject *pop_and_print_BAD(PyObject *pList) {
    PyObject *pLast;

    pLast = PyList_GetItem(pList, PyList_Size(pList) - 1);
    fprintf(stdout, "Ref count was: %zd\n", pLast->ob_refcnt);
    do_something(pList);
    fprintf(stdout, "Ref count now: %zd\n", pLast->ob_refcnt);
    PyObject_Print(pLast, stdout, 0);
    fprintf(stdout, "\n");
    Py_RETURN_NONE;
}
```
这里PLast是一个borrowed reference，这段代码看起来似乎没有问题，但让我们再仔细分析，`pList`拥有对它的对象的所有引用，所以在`do_something`中可能释放任何元素的引用，当它释放了所有元素的引用后，`PLast`是否还有效取决于最后一个元素是否还有别的引用。例如`do_something`可能如下：
```C
void do_something(PyObject *pList) {
    while (PyList_Size(pList) > 0) {
        PySequence_DelItem(pList, 0);
    }
}
```
那么，调用这个函数会发生什么事情？下面是一些例子(`pop_and_pring_BAD`被映射为`cPyRefs.popBAD`)：  
(1) 调用如下代码时，引用计数完全错误了，但是由于内存没有被改写，所以打印最后一个元素貌似是正确的。
```
>>> l = ["Hello", "World"]
>>> cPyRefs.popBAD(l)       # l will become empty
Ref count was: 1
Ref count now: 4302027608
'World'
```
(2) 以下代码出现了段错误，这个错误就比较明显了。
```
>>> l = ['abc' * 200]
>>> cPyRefs.popBAD(l)
Ref count was: 1
Ref count now: 2305843009213693952
Segmentation fault: 11
```
(3) 当调用下面的代码时，问题似乎又不见了,因为最后一个元素有额外的引用。
```
>>> l = ["Hello", "World"]
>>> a = l[-1]
>>> cPyRefs.popBAD(l)
Ref count was: 2
Ref count now: 1
'World'
```
上面这个例子的错误很难被发现，因为这个C函数的正确性依赖于调用者是否拥有额外的引用以及`do_something`的操作。当然，我们知道了引起问题的原因，解决方案也很简单，**用borrowed references时，如果你对对象感兴趣，你就应该为引用计数加1，然后在不用的时候再减1**。
```
static PyObject *pop_and_print_BAD(PyObject *pList) {
    PyObject *pLast;

    pLast = PyList_GetItem(pList, PyList_Size(pList) - 1);
    Py_INCREF(pLast);       /* Prevent pLast being deallocated. */
    /* ... */
    do_something(pList);
    /* ... */
    Py_DECREF(pLast);       /* No longer interested in pLast, it might     */
    pLast = NULL;           /* get deallocated here but we shouldn't care. */
    /* ... */
    Py_RETURN_NONE;
}
```

### 总结

在本文中，我们首先在第一节和第二节简单介绍了写Python C 扩展的方法和C调用Python的方法，然后在第三节，我们重点介绍了CPython API中的引用计数，以及引用计数中容易出现的内存泄露和非法内存访问问题，总的来说，几个比价重要的结论如下：  

* 大部分返回引用的函数都会将这个引用的拥有权转移到函数调用者,但`PyTuple_GetItem()`,`PyList_GetItem()`,`PyDIct_GetItem()`和`PyDict_GetItemString()`返回的是**borrowed Reference**；
* 在你将一个对象的引用传递进另一个函数时，一般来说这个函数会从你借这个引用，但`PyTuple_SetItem()`和`PyList_SetItem()`们会从你这偷取引用(**steal reference**)；
* 不要将返回New Reference的函数调用作为临时变量传递给一个函数的形参，例如`PyNumber_Subtract(PyLong_FromLong(a), PyLong_FromLong(b))`，会引起内存泄露；
* 用borrowed references时，如果你对对象感兴趣，你就应该为引用计数加1，然后在不用的时候再减1


#### 参考文献
1. [distutils官方文档](https://docs.python.org/2/distutils/index.html)
2. 一个python tutorial，直观的讲解怎么写C扩展并编译: [Python - Extension Programming with C](https://www.tutorialspoint.com/python/python_further_extensions.htm)
3. Python官方文档：[Extending Piython with C or C++](https://docs.python.org/2/extending/extending.html#intermezzo-errors-and-exceptions)
4. Python/C API，讲解Python 对象的设计层次，初始化，引用计数等：[Python/C API Reference Manual](https://docs.python.org/2/c-api/index.html)
5. C 调用 Python: [Calling a python method from C/C++, and extracting its return value](https://stackoverflow.com/questions/3286448/calling-a-python-method-from-c-c-and-extracting-its-return-value)
6. 一个对borrow和steal reference的回答：[Python C-API functions that borrow and steal references](https://stackoverflow.com/questions/10247779/python-c-api-functions-that-borrow-and-steal-references/10250720)
7. 详解Python reference以及可能出现的问题：[PyObjects and Reference Counting](http://pythonextensionpatterns.readthedocs.io/en/latest/refcount.html)