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
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT1.png" />  
上图中 testqd.hprof 文件是原始的Heap Dump文件，zip文件是生成的html形式的报告文件。  
打开之后，主界面如下所示：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT2.png" />  
接下来介绍界面中常用到的功能：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT3.png" />  

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
#### Duplicate Classes 
列出被加载多次的类，结果按类加载器进行分组，目标是加载同一个类多次被类加载器加载。使用该工具很容易找到部署应用的时候使用了同一个库的多个版本。
#### OQL
MAT提供了一个对象查询语言（OQL），跟SQL语言类似，将类当作表、对象当作记录行、成员变量当作表中的字段。通过OQL可以方便快捷的查询一些需要的信息，是一个非常有用的工具。
#### Thread Overview
此工具可以查看生成Heap Dump文件的时候线程的运行情况，用于线程的分析。
#### Run Expert System Test
可以查看分析完成的HTML形式的报告，也可以打开已经产生的分析报告文件，子菜单项如下图所示：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT4.png" />   
常用的主要有Leak Suspects和Top Components两种报告：
- Leak Suspects 可以说是非常常用的报告了，该报告分析了 Heap Dump并尝试找出内存泄漏点，最后在生成的报告中对检测到的可疑点做了详细的说明；
- Top Components 列出占用总堆内存超过1%的对象。  

#### Open Query Browser
提供了在分析过程中用到的工具，通常都集成在了右键菜单中，在后面具体举例分析的时候会做详细的说明。如下图：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT5.png" />  
#### Find Object by address
通过十六进制的地址查找对应的对象  

### 基础概念
#### Shallow Heap 和 Retained Heap
`Shallow Heap`表示对象本身占用内存的大小，不包含对其他对象的引用，也就是对象头加成员变量（不是成员变量的值）的总和。  
`Retained Heap`是该对象自己的`Shallow Heap`，并加上从该对象能直接或间接访问到对象的`Shallow Heap`之和。换句话说，`Retained Heap`是该对象GC之后所能回收到内存的总和。

#### GC Roots和Reference Chain
JVM在进行GC的时候是通过使用可达性来判断对象是否存活，通过GC Roots（GC根节点）的对象作为起始点，从这些节点开始进行向下搜索，搜索所走过的路径成为Reference Chain（引用链），当一个对象到GC Roots没有任何引用链相连（用图论的话来说就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。  

#### Histogram（直方图）视图
点击工具栏上的 Histogram图标可以打开Histogram（直方图）视图，可以列出每个类产生的实例数量，以及所占用的内存大小和百分比。主界面如下图所示：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT6.png" />   
图中Shallow Heap 和 Retained Heap分别表示对象自身不包含引用的大小和对象自身并包含引用的大小，具体请参考下面 Shallow Heap 和 Retained Heap 部分的内容。默认的大小单位是 Bytes，可以在 Window - Preferences 菜单中设置单位，图中设置的是MB。  
  
通过直方图视图可以很容易找到占用内存最多的几个类（通过Retained Heap排序），还可以通过其他方式进行分组（Group result by...）。  
  
如果存在内存溢出，时间久了溢出类的实例数量或者内存占比会越来越多，排名也越来越靠前。可以点击工具类上的图标进行对比，通过多次对比不同时间点下的直方图对比就很容易把溢出的类找出来。  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT7.png" />   

#### Dominator Tree视图
点击工具栏上的图标可以打开Dominator Tree（支配树）视图，在此视图中列出了每个对象（Object Instance）与其引用关系的树状结构，同时包含了占用内存的大小和百分比。  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT8.png" />   
通过Dominator Tree视图可以很容易的找出占用内存最多的几个对象（根据Retained Heap或Percentage排序），和Histogram类似，可以通过不同的方式进行分组显示。

#### 定位溢出源
Histogram视图和Dominator Tree视图的角度不同，前者是基于类的角度，后者是基于对象实例的角度，并且可以更方便的看出其引用关系。  
  
首先，在两个视图中找出疑似溢出的对象或者类（可以通过Retained Heap排序，并且可以在Class Name中输入正则表达式的关键词只显示指定的类名），然后右键选择Path To GC Roots（Histogram中没有此项）或Merge Shortest Paths to GC Roots，然后选择 exclude all phantom/weak/soft etc. reference：  
<img src="https://github.com/Faep1008/Study-Note/blob/master/notes/20200702/img/MAT9.png" />   
GC Roots意为GC根节点，其含义见上面的 GC Roots和Reference Chain 部分，后面的 exclude all phantom/weak/soft etc. reference 意思是排除虚引用、弱引用和软引用，即只剩下强引用，因为除了强引用之外，其他的引用都可以被JVM GC掉，如果一个对象始终无法被GC，就说明有强引用存在，从而导致在GC的过程中一直得不到回收，最终就内存溢出了。  
  
通过结果就可以很方便的定位到具体的代码，然后分析是什么原因无法释放该对象，比如被缓存了或者没有使用单例模式等等。  
