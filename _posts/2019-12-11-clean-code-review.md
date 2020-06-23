---
layout: post
title:  "代码整洁之道读书笔记"
date:   2019-12-11 00:00:00
categories: 读书笔记
tags: 技术书籍读书笔记
---

* content
{:toc}




## 为什么要保持代码整洁？
**高大上**：
1. 软件质量，不但依赖于架构及项目管理，而且与代码质量紧密相关；而代码质量与其整洁度成正比。  
     软件质量 <=> 代码质量 <=> 整洁度。  
2. 如果将软件比作一座宏大的建筑的话，那么宏大建筑中最细小的部分，比如关不紧的门、有点儿没铺平的地板，甚至是凌乱的桌面，都会将整个大局的魅力毁灭殆尽。这就是整洁代码之所系。  

**贴合实际**：  
1. 代码混乱的代价：随着混乱的增加，团队生产力持续下降，趋向于零。（混乱增加，生产力下降 -> 管理层增加人手，期望提升生产力 -> 新人不熟悉系统的设计，制造更多的混乱）。  
![produce vs time](https://img-blog.csdnimg.cn/20200623141349471.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdHp6aGFuZw==,size_16,color_FFFFFF,t_70)  
2. 糟糕代码毁掉应用的例子  
20世纪80年代末，有家公司写了个很流行的杀手应用，许多专业人士都买来用。 ->  发布周期开始拉长，缺陷总是不能修复，装载时间越来越久，崩溃的几率也越来越大。 -> 客户沮丧地关掉那个程序，从此不再用它。  
原因：  
• 赶着推出产品，代码写的乱七八糟。  
• 特性越加越多，代码也越来越烂，最后再也没法管理这些代码。  
• 糟糕的代码毁了这家公司。   

**为什么会写糟糕代码**？     
1. 每个人都为糟糕的代码所困扰过，那么，为什么要写糟糕的代码呢？  
• 追求效率，想要快点完成。（回家老婆孩子热炕头）  
• 要赶时间。或许你觉得自己要干好所需工作的时间不够，假使花时间清理代码，老板就会大发雷霆  
• 或许你只是不耐烦再搞这套程序，期望早点结束。  
• 或许你看了看自己承诺要做的其他事，意识到得赶紧弄完手上的东西，好接着做下一件工作。  
2. 自己亲手造成的混乱，没有及时清理？  
看到自己的烂程序居然能运行，然后断言能运行的烂程序总比什么都没有强。 -> 我们都曾经说过有朝一日再回头清理。 -> 勒布朗(LeBlanc)法则：稍后等于永不（Later equals never）。   

**什么是整洁代码**？  
• 易读  
• 没有重复代码  
• 尽量少的依赖关系  
• 每个函数、每个类、每个模块只做好一件事。  
![good code vs bad code](https://img-blog.csdnimg.cn/20200623141349530.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdHp6aGFuZw==,size_16,color_FFFFFF,t_70)  

## 第2章 有意义的命名 

### 1. 名副其实    
变量、函数或类的名称应该已经答复了所有的大问题。它该告诉你，它为什么会存在，它做什么事，应该怎么用。如果名称需要注释来补充，那就不算是名副其实。

```java
int d;   // 消逝的时间，以日计，d什么也没说明
// 应该选择指明了计量对象和计量单位的名称
int elapsedTimeInDays;
int daysSinceCreation;
int daysSinceModification;
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<int[]>();
    for(int[] x : theList) {
        if(x[0] == 4) {
            list1.add(x);
        }
    }
    return list1;
}
```
• theList中是什么类型的东西？    
• theList零下标条目的意义是什么？  
• 值4的意义是什么？  
• 我怎么使用返回的列表？  

```java
public List<int[]> getFlaggedCells() {
    List<int[]> flaggedCells = new ArrayList<int[]>();
    for(int[] cell : gameBoard) {
        if(cell[STATUS_VALUE] == FLAGGED) {
            flaggedCells.add(cell);
        }
    }
    return flaggedCells;
}
```

### 2. 做有意义的区分  
getActiveAccount();  
getActiveAccounts();  
getActiveAccountInfo();  
根本不知道调用哪个函数。  

### 3. 使用可搜索的名称
找MAX_CLASSES_PER_SUTUDENT很容易，但想找数字7就麻烦了。

### 4. 类名和对象名应该是名词或名词短语，方法名应该是动词或动词短语

## 第3章 函数  
### 1. 短小
函数的第一规则是要短小，第二条规则是还要更短小。(集团代码规约认为函数体行数应小于80行)  

```java
public static String testableHtml(
     PageData pageData,
     boolean includeSuiteSetup
   ) throws Exception {
     WikiPage wikiPage = pageData.getWikiPage();
     StringBuffer buffer = new StringBuffer();
     if (pageData.hasAttribute("Test")) {
       if (includeSuiteSetup) {
         WikiPage suiteSetup =
           PageCrawlerImpl.getInheritedPage(
               SuiteResponder.SUITE_SETUP_NAME, wikiPage
           );
         if (suiteSetup != null) {
           WikiPagePath pagePath =
             suiteSetup.getPageCrawler().getFullPath(suiteSetup);
           String pagePathName = PathParser.render(pagePath);
           buffer.append("!include -setup .")
                 .append(pagePathName)
                 .append("\n");
         }
       }
       WikiPage setup =
         PageCrawlerImpl.getInheritedPage("SetUp", wikiPage);
       if (setup != null) {
         WikiPagePath setupPath =
           wikiPage.getPageCrawler().getFullPath(setup);
         String setupPathName = PathParser.render(setupPath);
         buffer.append("!include -setup .")
               .append(setupPathName)
               .append("\n");
       }
     }
     buffer.append(pageData.getContent());
     if (pageData.hasAttribute("Test")) {
       WikiPage teardown =
         PageCrawlerImpl.getInheritedPage("TearDown", wikiPage);
       if (teardown != null) {
         WikiPagePath tearDownPath =
           wikiPage.getPageCrawler().getFullPath(teardown);
         String tearDownPathName = PathParser.render(tearDownPath);
         buffer.append("\n")
               .append("!include -teardown .")
               .append(tearDownPathName)
               .append("\n");
       }
       if (includeSuiteSetup) {
         WikiPage suiteTeardown =
           PageCrawlerImpl.getInheritedPage(
                   SuiteResponder.SUITE_TEARDOWN_NAME,
                   wikiPage
           );
         if (suiteTeardown != null) {
           WikiPagePath pagePath =
             suiteTeardown.getPageCrawler().getFullPath (suiteTeardown);
           String pagePathName = PathParser.render(pagePath);
           buffer.append("!include -teardown .")
                 .append(pagePathName)
                 .append("\n");
         }
      }
    }
    pageData.setContent(buffer.toString());
    return pageData.getHtml();
   }
```
搞懂整个函数了么？大概没有。有太多事发生，有太多不同层级的抽象，奇怪的字符串和函数调用，用标识来控制的if语句等，不一而足。  
不过，只做几个简单的方法抽离和重命名工作，加上一点点重构，就能在9行代码之内搞定。  

```java
public static String renderPageWithSetupsAndTeardowns(
    PageData pageData, boolean isSuite) throws Exception {
    boolean isTestPage = pageData.hasAttribute("Test");
    if(isTestPage) {
        WikiPage testPage = pageData.getWikiPage();
        StringBuffer newPageContent = new StringBuffer();
        includeSetupPages(testPage, newPageContent, isSuite);
        newPageContent.append(pageData.getContent());
        includeTeardownPages(testPage, newPageContent, isSuite);
        pageData.setContent(newPageContent.toString());
    }
    return pageData.getHtml();
}
```
还可以更加短小：

```java
public static String renderPageWithSetupsAndTeardowns(
    PageData pageData, boolean isSuite) throws Exception {
    if(isTestPage(pageData)) {
        includeSetupAndTeardownPages(pageData, isSuite);
    }
    return pageData.getHtml();
}
```

### 2. 函数应该做一件事。做好这件事。只做这一件事

### 3. 使用描述性的名称
别害怕长名称，长而具有描述性的名称，要比短而令人费解的名称好。

### 4. 函数参数
• 最理想的参数数量是零，其次是一，再次是二，应尽量避免三参数函数。参数越多越难懂，调用时越容易出错。  

```java
// 读到message，错以为它是expected？
// 搞不清expected和actual的顺序
assertEquals(message, expected, actual) 
```
• 不传标识参数。向函数传入布尔值简直就是骇人听闻的做法，大声宣布本函数不止做一件事。   
• 避免使用输出参数,使用返回结果而不是修改入参。   
• 如果函数需要两个、三个或三个以上参数，就说明其中一些参数应该封装为类了。   

### 5. 使用异常替代返回错误码  
返回错误码时，就是在要求调用者立刻处理错误。

```java
if(deletePage(page) == E_OK) {
    if(registry.deleteReference(page.name) == E_OK) {
        if(configKeys.deleteKey(pange.name.makeKey()) == E_OK) {
            logger.log("page deleted");
        } else {
            logger.log("configKey not deleted");
        }
    } else {
        logger.log("deleteReference from registry failed");
    }
} else {
    logger.log("delete failed");
    return E_ERROR;
}
```

另一方面，如果使用异常替代返回错误码，错误处理代码就能从主路径代码中分离出来，得到简化：

```java
try {
    deletePage(page);
    registry.deleteReference(page.name);
    configKeys.deleteKey(page.name.makeKey());
} catch (Exception e) {
    logger.log(e.getMessage());
}
```

### 6. 别重复自己   
重复可能是软件中一切邪恶的根源，许多原则与实践规则都是为了控制与消除重复而创建。

### 7. 如何写出这样的函数   
• 写代码和写别的东西很像。在写论文或文章时，你先想什么就写什么，然后再打磨它。初稿也许粗陋无序，你就斟酌推敲，直至达到你心目中的样子。  
• 写函数时，一开始都冗长而复杂。有太多缩进和嵌套循环。有过长的参数列表。名称也是随意取的，也会有重复的代码。不过配上一套单元测试，覆盖每行丑陋的代码。  
• 然后打磨这些代码，分解函数、修改名称、消除重复。缩短和重新安置方法。有时还要拆散类。同时保持测试通过。  
• 最后，遵循本章列出的规则，组装好这些函数。  
并不从一开始就按照规则写函数；没人做得到。  

## 第4章 注释

### 1. 注释是把双刃剑    
什么也比不上放置良好的注释来得有用。什么也不会比乱七八糟的注释更有本事搞乱一个模块。什么也不会比陈旧、提供错误信息的注释更有破坏性。（注释是把双刃剑）
### 2. 注释是用来弥补代码表达能力的不足的
注释不能美化糟糕的代码，用代码来阐述工作。  
### 3. 作者极力贬低注释
因为注释会撒谎，他认为程序员不能坚持维护注释。尽管有时也需要注释，我们也该多花心思尽量减少注释量。
### 4. 好注释
• 法律信息  

```java
// Copyright (C) 2003,2004,2005 by Object Mentor, Inc. All rights reserved
```
• 提供信息的注释  

```java
// Returns an instance of the Responder being tested.
protected abstract Responder responderInstance();
```

• 对意图的解释  

```java
// This is our best attempt to get a race condition
// by creating large number of threads
for(int i =0; i<25000; i++) {
    Thread thread = new Thread(widgetBuilderThread);
    thread.start();
}
```
• 警示  

```java
public static SimpleDateFormat makeStandardHttpDateFormat() {
    // SimpleDateFormat is not thread safe
    // so we need to create each instance independently.
    SimpleDateFormat df = new SimpleDateFormat("EEE, dd MM yyy HH:mm:ss z");
    df.setTimeZone(TimeZone.getTimeZone("GTM"));
    return df;
}
```
• TODO 注释  
• 公共API中的Javadoc  

### 7. 坏注释  

• 多余的注释

```java
// Utility method that returns when this.closed is true. Throws an exception if
// the timeout is reached
public synchronized void waitForClose(final long timeoutMillis) Throws Exception {
    if(!closed) {
        wait(timeoutMillis);
        if(!closed) {
            throw new Exception("MockResponseSender could not be closed");
        }
    }
}
```
这段注释并不能比代码本身提供更多的信息，读它也并不比读代码更容易。  
• 循轨式注释  
所谓每个函数都要有Javadoc或每个变量都要有注释的规矩全然是愚蠢可笑的。  
• 日志式注释 

```java
* Changes (from 11-oct-2011)
* --------------------------
* 11-Oct-2001: Re-organised the class and moved it to new package com.jrefiner.date
* 05-Nov-2001: Added a getDescription() method, and eliminated NotableDate class
```
• 位置标记  
有时，程序员喜欢在源代码中标记某个特别位置。  

```java
// Actions ////////////////////////////////////////////////////////
```
• 注释掉的代码  
直接把代码注释掉是讨厌的做法，别这么干。  

```java
InputStreamResponse response = new InputStreamResponse();
response.setBody(formatter.getResultStream(), formatter.getByteCount());
// InputStream resultStream = formatter.getResultStream();
// StreamReader reader = new StreamReader(resultStream);
// response.setContent(reader.read(formatter.getByteCount());
```
其他人不敢删除注释掉的代码，他们会想，代码依然放在那儿，一定有其原因，而且这段代码很重要，不能删除。  

## 第5章 格式  
### 1. 团队格式规范
如果你在团队中工作，则团队应该一致同意采用一套简单的格式规则，所有成员都要遵从。
### 2. 垂直格式
被调用的函数应该放在执行调用的函数下面。这样就建立了一种自顶向下贯穿源代码模块的良好信息流。
### 3. 变量声明应尽可能靠近其使用位置
### 4. 实体变量应该在类的顶部声明


## 第6章 对象和数据结构
### 1. 对象和数据结构. 
数据结构：暴露其数据，没有提供有意义的函数  

```java
public class Point {
    public double x;
    public double y;
}
```
对象：把数据隐藏于抽象之后，暴露操作数据的函数  

```java
public class Point {
    private double x;
    private double y;
    double getX();
    double getY();
    void setCartesian(double x, double y);
}
```
### 2. 过程式代码和面向对象代码  
过程式代码：

```java
public class Square {
    public Point topLeft;
    public double side;
}
public class Rectangle {
    public Point topLeft;
    public double height;
    public double width;
}
public class Circle {
    public Point center;
    public double radius;
}
public class Geometry {
    public final double PI = 3.14159265358;
    
    public double area(Object shape) throws NoSuchShapeException {
        if(shape instanceof Square) {
            Square s = (Square)shape;
            return s.side * s;
        } else if(shape instanceof Rectangle) {
            Rectanle r = (Rectanle)shape;
            return r.height * r.width;
        } else if(shape instanceof Circle) {
            Circle c = (Circle)shape;
            return PI * c.radius * c.radisu;
        }
        throw new NoSuchShapeException();
    }
}
```
面向对象代码：  

```java
public class Square implements Shape {
    private Point topLeft;
    private double side;
    
    public double area() {
        return side * side;
    }
}
public class Rectangle implements Shape {
    private Point topLeft;
    private double height;
    private double width;
    
    public double area() {
        return height * width;
    }
}
public class Circle implements Shape {
    private Point center;
    private double radius;
    public final double PI = 3.14159265358;
    
    public double area() {
        return PI * radius * radius;
    }
}
```
### 3. 过程式代码 vs 面向对象代码
过程式代码：  
• 添加新函数： 形状类根本不会受到影响  
• 添加新类：得修改Geometry中的所有函数来处理它  
面向对象代码：  
• 添加新函数：所有的形状类都得做修改  
• 添加新类：现有的函数一个也不会收到影响  
这两种定义的本质，是截然对立的。过程式代码（使用数据结构的代码）便于在不改动既有数据结构的前提下添加新函数；面向对象代码便于在不改动既有函数的前提下添加新类。  
一切都是对象只是一个传说，有时候你真的想要在简单数据结构上做一些过程式的操作。  

### 4. 得墨忒耳律（The Law of Demeter)

类C的方法f只应该调用以下对象的方法：   
• C  
• 由f创建的对象；  
• 作为参数传递给f的对象；  
• 由C的实体变量持有的对象。  
方法不应调用由任何函数返回的对象的方法。  

```java
final String outputDir = ctx.getOptions().getScrachDir().getAbsolutePath();
```


## 第7章 错误处理
### 1. 使用异常而非返回码  
在很久以前，许多语言都不支持异常（C语言）。这些语言处理和汇报错误的手段都有限，你要么设置一个错误标识，要么返回给调用者检查错误的代码。  
这类手段的问题在于，它们搞乱了调用者代码，调用者必须在调用之后即刻检查错误。不幸的是，这个步骤很容易被遗忘。  

### 2. 使用不可控(unchecked)异常  
可控异常的代价就是违反开放/闭合原则。如果你在方法中抛出可控异常，而catch语句在三个层级之上，你就得在catch语句和抛出异常处之间的每个方法签名中声明该异常。这意味着对软件中较低层级的修改，都将波及较高层级的签名。  

### 3. 别返回null值  
返回null值，基本上是在给自己增加工作量，也是在给调用者添乱。只要有一处没有检查null值，应用程序就会失控。  

```java
public void registerItem(Item item) {
    if (item != null) {
        ItemRegistry registry = peristentStore.getItemRegistry();
        if (registry != null) {
            Item existing = registry.getItem(item.getID());
            // 没有检查null，会发生什么事？
            if(existing.getBillingPeriod().hasRetailOwner()) {
                existing.register(item);
            }
        }
    }
}    
```

返回null值，不如抛出异常，或是返回特例对象。  

## 第8章 边界  
学习性测试：编写测试来遍览和理解第三方代码。  

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    logger.info("hello");
}
```

发生错误，告诉我们需要用Appender -> 阅读文档，发现有个ConsoleAppender，添加ConsoleAppender  

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    ConsoleAppender appender = new ConsoleAppender();
    logger.addAppender(appender);
    logger.info("hello");
}
```
发现Appender没有输出流。Google得到帮助后添加patternLayout:  

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    logger.removeAllAppenders();
    ConsoleAppender appender = new ConsoleAppender(
        new PatternLayout("%p %t %m%n"), ConsoleAppender.SYSTEM_OUT
    );
    logger.addAppender(appender);
    logger.info("hello");
}
```
正确，输出hello到控制台！


## 第9章 单元测试
### 1. 保持测试整洁  
脏测试等同于——如果不是坏于的话——没测试。测试必须随生产代码的演进而修改，测试越脏，就越难修改。 测试代码和生产代码一样重要 。  
如果测试不能保持整洁，你就会失去它们。没有了测试，你就会失去保证生产代码可扩展的一切要素。正是单元测试让你的代码可扩展、可维护、可复用！  

### 2. 整洁测试的三个要素：可读性，可读性和可读性  

```java
public void testGetPageHieratchyAsXml() throws Exception {
  crawler.addPage(root, PathParser.parse("PageOne"));
  crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
  crawler.addPage(root, PathParser.parse("PageTwo"));
  request.setResource("root");
  request.addInput("type", "pages");
  Responder responder = new SerializedPageResponder();
  SimpleResponse response =
    (SimpleResponse) responder.makeResponse(new FitNesseContext(root), request);
  String xml = response.getContent();
  assertEquals("text/xml", response.getContentType());
  assertSubString("<name>PageOne</name>", xml);
  assertSubString("<name>PageTwo</name>", xml);
  assertSubString("<name>ChildOne</name>", xml);
}

public void testGetPageHieratchyAsXmlDoesntContainSymbolicLinks() throws Exception {
  WikiPage pageOne = crawler.addPage(root, PathParser.parse("PageOne"));
  crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
  crawler.addPage(root, PathParser.parse("PageTwo"));
  PageData data = pageOne.getData();
  WikiPageProperties properties = data.getProperties();
  WikiPageProperty symLinks = properties.set(SymbolicPage.PROPERTY_NAME);
  symLinks.set("SymPage", "PageTwo");
  pageOne.commit(data);
  request.setResource("root");
  request.addInput("type", "pages");
  Responder responder = new SerializedPageResponder();
  SimpleResponse response =
    (SimpleResponse) responder.makeResponse(new FitNesseContext(root), request);
  String xml = response.getContent();
  assertEquals("text/xml", response.getContentType());
  assertSubString("<name>PageOne</name>", xml);
  assertSubString("<name>PageTwo</name>", xml);
  assertSubString("<name>ChildOne</name>", xml);
  assertNotSubString("SymPage", xml);
}
```
测试很难读懂：  
• 有数量恐怖的重复代码调用addPage 和 addSubString。  
• 代码中充满了干扰测试表达力的细节。  
重构：  

```java
public void testGetPageHierarchyAsXml() throws Exception {
  makePages("PageOne", "PageOne.ChildOne", "PageTwo");
  submitRequest("root", "type:pages");
  assertResponseIsXML();
  assertResponseContains(
    "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>");
}

public void testSymbolicLinksAreNotInXmlPageHierarchy() throws Exception {
  WikiPage page = makePage("PageOne");
  makePages("PageOne.ChildOne", "PageTwo");
  addLinkTo(page, "PageTwo", "SymPage");
  submitRequest("root", "type:pages");
  assertResponseIsXML();
  assertResponseContains(
    "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>");
  assertResponseDoesNotContain("SymPage");
}
```

## 第10章 类
### 1. 类应该短小
类的第一条规则是类应该短小。第二条规则是还要更短小。
### 2. 单一权责原则（SRP）  
类或模块应有且只有一条加以修改的理由。

## 第11章 系统
### 1. 将系统的构造与使用分开  
软件系统应将启动过程和启始过程之后的运行时逻辑分离开。
### 2. 依赖注入
有一种强大的机制可以实现分离构造与使用，那就是依赖注入（Dependency Injection）  

## 第12章 迭进
通过跌进设计达到整洁目的，简单设计的4条规则：  
1. 运行所有测试
2. 不可重复
3. 表达程序员的意图    
• 通过选用好名称来表达  
• 通过保持函数和类尺寸短小来表达  
• 通过采用标准命名法来表达  
• 编写良好的测试单元也具有表达性  
4. 尽可能减少类和方法的数量