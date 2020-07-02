# JVM Heap Dump（堆转储文件）的生成和MAT的使用
## JVM Heap Dump（堆转储文件）的生成
Heap Dump记录了JVM中堆内存运行的情况。可以通过以下几种方式生成Heap Dump文件：
### 使用 jmap 命令生成
jmap 命令是JDK提供的用于生成堆内存信息的工具，可以执行下面的命令生成Heap Dump：
```shell
jmap -dump:live,format=b,file=heap-dump.hprof <pid>
```
其中的pid是JVM进程的id，heap-dump.hprof是生成的文件名称，在执行命令的目录下面。推荐此种方法。
### 在JVM中增加参数生成
在JVM的配置参数中可以添加 `-XX:+HeapDumpOnOutOfMemoryError` 参数，当应用抛出 `OutOfMemoryError` 时自动生成dump文件；
在JVM的配置参数中添加 `-Xrunhprof:head=site` 参数，会生成`java.hprof.txt` 文件，不过这样会影响JVM的运行效率，不建议在生产环境中使用。
### 使用 JConsole 生成
JConsole是JDK提供的一个基于GUI查看JVM系统信息的工具，既可以管理本地的JVM，也可以管理远程的JVM，可以通过 dumpHeap 按钮生成 Heap Dump文件。

## 常见的Heap Dump文件分析工具
JVM Heap Dump文件可以使用常用的分析工具如下：
### jhat
jhat 是JDK自带的用于分析JVM Heap Dump文件的工具，使用下面的命令可以将堆文件的分析结果以HTML网页的形式进行展示：
```
jhat <heap-dump-file>
```
其中 heap-dump-file 是文件的路径和文件名，可以使用 `-J-Xmx512m` 参数设置命令的内存大小。执行成功之后显示如下结果：
```
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```
这个时候访问 `http://localhost:7000/` 就可以看到结果了。

### Eclipse Memory Analyzer(MAT)
Eclipse Memory Analyzer(MAT)是Eclipse提供的一款用于Heap Dump文件的工具，操作简单明了，下面将详细进行介绍。
#### 主界面
第一次打开因为需要分析dump文件，所以需要等待一段时间进行分析，分析完成之后dump文件目录下面的文件信息如下：
=1111==============================================
上图中 testqd.hprof 文件是原始的Heap Dump文件，zip文件是生成的html形式的报告文件。
打开之后，主界面如下所示：
=2222==============================================
接下来介绍界面中常用到的功能：
=3333==============================================

#### Overview
Overview视图，即概要界面，显示了概要的信息，并展示了MAT常用的一些功能。  

- Details 显示了一些统计信息，包括整个堆内存的大小、类（Class）的数量、对象（Object）的数量、类加载器（Class Loader)的数量。
- Biggest Objects by Retained Size 使用饼图的方式直观地显示了在JVM堆内存中最大的几个对象，当光标移到饼图上的时候会在左边Inspector和Attributes窗口中显示详细的信息。
- Actions 这里显示了几种常用到的操作，算是功能的快捷方式，包括 Histogram、Dominator Tree、Top Consumers、Duplicate Classes，具体的含义和用法见下面；
- Reports 列出了常用的报告信息，包括 Leak Suspects和Top Components，具体的含义和内容见下；
- Step By Step 以向导的方式引导使用功能。
  
#### Histogram
直方图，可以查看每个类的实例（即对象）的数量和大小。
#### Dominator Tree
支配树，列出Heap Dump中处于活跃状态中的最大的几个对象，默认按 retained size进行排序，因此很容易找到占用内存最多的对象。
#### Top Consumers 
按类、类加载器和包分别进行查询，并以饼图的方式列出最大的几个对象。
#### Duplicate Classes 列出被加载多次的类，结果按类加载器进行分组，目标是加载同一个类多次被类加载器加载。使用该工具很容易找到部署应用的时候使用了同一个库的多个版本。
#### OQL
MAT提供了一个对象查询语言（OQL），跟SQL语言类似，将类当作表、对象当作记录行、成员变量当作表中的字段。通过OQL可以方便快捷的查询一些需要的信息，是一个非常有用的工具。
#### Thread Overview
此工具可以查看生成Heap Dump文件的时候线程的运行情况，用于线程的分析。
#### Run Expert System Test
可以查看分析完成的HTML形式的报告，也可以打开已经产生的分析报告文件，子菜单项如下图所示：  
==4444====================================  
常用的主要有Leak Suspects和Top Components两种报告：
- Leak Suspects 可以说是非常常用的报告了，该报告分析了 Heap Dump并尝试找出内存泄漏点，最后在生成的报告中对检测到的可疑点做了详细的说明；
- Top Components 列出占用总堆内存超过1%的对象。  

#### Open Query Browser
提供了在分析过程中用到的工具，通常都集成在了右键菜单中，在后面具体举例分析的时候会做详细的说明。如下图：  
===55555=====================================
#### Find Object by address
通过十六进制的地址查找对应的对象
