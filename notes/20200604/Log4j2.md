# Log4j2
## 简介
log4j2相对于log4j 1.x有了脱胎换骨的变化，其官网宣称的优势有多线程下10几倍于log4j 1.x和logback的高吞吐量、可配置的审计型日志、基于插件架构的各种灵活配置等。

## 基础配置
普通Java项目手动添加Jar包
```
log4j-api-2.5.jar
log4j-core-2.5.jar
```
Maven项目pom.xml
```xml
<dependencies>
	<dependency>
		<groupId>org.apache.logging.log4j</groupId>
		<artifactId>log4j-api</artifactId>
		<version>2.5</version>
	</dependency>
	<dependency>
		<groupId>org.apache.logging.log4j</groupId>
		<artifactId>log4j-core</artifactId>
		<version>2.5</version>
	</dependency>
</dependencies>
```
测试代码
```java
public static void main(String[] args) {
	Logger logger = LogManager.getLogger(LogManager.ROOT_LOGGER_NAME);
	logger.trace("trace level");
	logger.debug("debug level");
	logger.info("info level");
	logger.warn("warn level");
	logger.error("error level");
	logger.fatal("fatal level");
}
```
运行后输出
```java
ERROR StatusLogger No log4j2 configuration file found. Using default configuration: logging only errors to the console.
20:37:11.965 [main] ERROR  - error level
20:37:11.965 [main] FATAL  - fatal level
```
可以看到log4j2先发了一句牢骚，抱怨没有找到配置文件什么的，不过还是输出了error和fatal两个级别的信息。  
`log4j2`默认会在`classpath`目录下寻找`log4j.json`、`log4j.jsn`、`log4j2.xml`等名称的文件，如果都没有找到，则会按默认配置输出，也就是输出到控制台。  
  
下面我们按默认配置添加一个log4j2.xml，添加到src根目录即可
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
    </Appenders>
    <Loggers>
        <Root level="error">
            <AppenderRef ref="Console" />
        </Root>
    </Loggers>
</Configuration>
```
重新执行测试代码，可以看到输出结果相同，但是没有再提示找不到配置文件。  
来看添加的配置文件log4j2.xml，以`Configuration`为根节点，有一个`status`属性，这个属性表示log4j2本身的日志信息打印级别。如果把status改为TRACE再执行测试代码，可以看到控制台中打印了一些log4j加载插件、组装logger等调试信息。后面会详细介绍。  

## 日志级别
日志级别从低到高分为：  
`TRACE` < `DEBUG` < `INFO` < `WARN` < `ERROR` < `FATAL`  
设为高级别等级时，低于设置的等级的日志不会打印；比如设置为WARN，则低于WARN的信息都不会输出。如果将log level设置在某一个级别上，那么比此级别优先级高的log都能打印出来。对于Loggers中的level的定义同样适用。  
`log4j`默认的优先级为`ERROR`或者`WARN`（实际上是`ERROR`）。

## Appender配置
`Appender`可以理解为日志的输出目的地。  
这里配置了一个类型为`Console`的`Appender`，也就是输出到控制台，`Console`节点中的`PatternLayout`定义了输出日志时的格式：  
- `%d{HH:mm:ss SSS}` 表示输出到毫秒的时间
- `%t` 输出当前线程的名称
- `%-5level` 输出日志级别 -5表示左对齐并且固定输出5个字符，如果不足右边补0
- `%logger` 输出logger名称 因为Root Logger没有名称所以没有输出
- `%msg` 日志文本
- `%n` 换行
- `%F` 输出所在的类文件名
- `%L` 输出行号
- `%M` 输出所在方法
- `%l` 输出语句所在的行数、包括类名、方法名、文件名、行数
- 最后是Logger的配置，这里只配置了一个Root Logger
  
配置文件
```xml
<Configuration status="WARN" monitorInterval="300">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
    </Appenders>
    <Loggers>
    	<Logger name="mylog" level="trace" additivity="false">
	    <AppenderRef ref="Console" />
	</Logger>
        <Root level="error">
            <AppenderRef ref="Console" />
        </Root>
    </Loggers>
</Configuration>
```
`additivity="false"`表示在该`logger`中输出的日志不会再延伸到父层`logger`。这里如果改为`true`，则会延伸到`Root Logger`，遵循`Root Logger`的配置也输出一次。  
注意根节点增加了一个`monitorInterval`属性，含义是每隔`300秒`重新读取配置文件，可以不重启应用的情况下修改配置，还是很好用的功能。  
`root`标签为`log`的默认输出形式，如果一个类的`log`没有在`loggers`中明确指定其输出`lever`与格式，那么就会采用`root`中定义的格式。  

## 自定义Appender
配置一个按时间和文件大小滚动的`RollingRandomAccessFile Appender`，名字真是够长，但不光只是名字长，相比`RollingFileAppender`有很大的性能提升，官网宣称是20-200%。  
`Rolling`的意思是当满足一定条件后，就重命名原日志文件用于备份，并从新生成一个新的日志文件。例如需求是每天生成一个日志文件，但是如果一天内的日志文件体积已经超过1G，就从新生成，两个条件满足一个即可。这在log4j 1.x原生功能中无法实现，在log4j2中就很简单了。  
配置如下：
```xml
<Configuration status="WARN" monitorInterval="300">
	<properties>
		<property name="LOG_HOME">D:/logs</property>
		<property name="FILE_NAME">mylog</property>
	</properties>
	<Appenders>
		<Console name="Console" target="SYSTEM_OUT">
			<PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
		</Console>
		<RollingRandomAccessFile name="MyFile"
			fileName="${LOG_HOME}/${FILE_NAME}.log"
			filePattern="${LOG_HOME}/$${date:yyyy-MM}/${FILE_NAME}-%d{yyyy-MM-dd HH-mm}-%i.log">
			<PatternLayout
				pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
			<Policies>
				<TimeBasedTriggeringPolicy interval="1" />
				<SizeBasedTriggeringPolicy size="10 MB" />
			</Policies>
			<DefaultRolloverStrategy max="20" />
		</RollingRandomAccessFile>
	</Appenders>
 
	<Loggers>
		<Logger name="mylog" level="trace" additivity="false">
			<AppenderRef ref="MyFile" />
		</Logger>
		<Root level="error">
			<AppenderRef ref="Console" />
		</Root>
	</Loggers>
</Configuration>
```
关键的字段解释：  
`<properties>`定义了两个常量方便后面使用。  
`RollingRandomAccessFile`的属性：  
`fileName`  指定当前日志文件的位置和文件名称  
`filePattern`  指定当发生Rolling时，文件的转移和重命名规则  
`SizeBasedTriggeringPolicy`  指定当文件体积大于size指定的值时，触发Rolling  
`DefaultRolloverStrategy`  指定最多保存的文件个数  

`TimeBasedTriggeringPolicy` 这个配置需要和`filePattern`结合使用，注意`filePattern`中配置的文件重命名规则是`${FILE_NAME}-%d{yyyy-MM-dd HH-mm}-%i`，最小的时间粒度是`mm`，即分钟，`TimeBasedTriggeringPolicy`指定的`size`是`1`，结合起来就是每1分钟生成一个新文件。如果改成`%d{yyyy-MM-dd HH}`，最小粒度为小时，则每一个小时生成一个文件。  

## 自定义配置文件位置
`log4j2`默认在`classpath`下查找配置文件，可以修改配置文件的位置。在非web项目中：
```java
public static void main(String[] args) throws IOException {
	File file = new File("D:/log4j2.xml");
	BufferedInputStream in = new BufferedInputStream(new FileInputStream(file));
	final ConfigurationSource source = new ConfigurationSource(in);
	Configurator.initialize(null, source);
	
	Logger logger = LogManager.getLogger("mylog");
}
```
如果是web项目，在web.xml中添加
```xml
<context-param>
	<param-name>log4jConfiguration</param-name>
	<param-value>/WEB-INF/conf/log4j2.xml</param-value>
</context-param>

<listener>
	<listener-class>org.apache.logging.log4j.web.Log4jServletContextListener</listener-class>
</listener>
```

## FAQ
### 如果说在一个类中定义一个静态成员变量logger，在多线程的环境下会不会出现线程安全问题？
不存在线程安全问题，logger本身就是一个单例,打印日志时先创建message(这个是在方法中实现的)。

## 日志框架比较
### 日志接口(slf4j)
`slf4j`是对所有日志框架制定的一种规范、标准、接口，并不是一个框架的具体实现，因为接口不能独立使用，需要和具体的日志框架实现配合使用（如`log4j`、`logback`）

### 日志实现（log4j、logback、log4j2）
- `log4j`是Apache实现的一个开源日志组件
- `logback`也是由`log4j`的设计者完成的，拥有更好的特性，用来取代`log4j`的一个日志框架，是`slf4j`的原生实现
- `log4j2`是`log4j 1.x`和`logback`的改进版，据说采用了一些新技术（无锁异步、等等），使得日志的吞吐量、性能比`log4j 1.x`提高10倍，并解决了一些死锁的bug，而且配置更加简单灵活。

### 为什么需要日志接口，直接使用具体的实现不就行了吗？
接口用于定制规范，可以有多个实现，使用时是面向接口的（导入的包都是`slf4j`的包而不是具体某个日志框架中的包），即直接和接口交互，不直接使用实现，所以可以任意的更换实现而不用更改代码中的日志相关代码。  
比如：slf4j定义了一套日志接口，项目中使用的日志框架是logback，开发中调用的所有接口都是slf4j的，不直接使用logback，调用是 自己的工程调用slf4j的接口，slf4j的接口去调用logback的实现，可以看到整个过程应用程序并没有直接使用logback，当项目需要更换更加优秀的日志框架时（如log4j2）只需要引入Log4j2的jar和Log4j2对应的配置文件即可，完全不用更改Java代码中的日志相关的代码`logger.info(“xxx”)`，也不用修改日志相关的类的导入的包（`import org.slf4j.Logger;import org.slf4j.LoggerFactory;`）  
  
==log4j、logback、log4j2都是一种日志具体实现框架，所以既可以单独使用也可以结合slf4j一起搭配使用==  

## Log4j2配置文件详解
log4j2.xml文件的配置大致如下：

- Configuration
  - properties
  - Appenders
    - Console
      - PatternLayout
    - File
    - RollingRandomAccessFile
    - Async
  - Loggers
    - Logger
    - Root
      - AppenderRef

解释：
- `Configuration`：根节点，有`status`和`monitorInterval`等属性  
①`status`的值有：trace”, “debug”, “info”, “warn”, “error” , “fatal”，用于控制`log4j2`日志框架本身的日志级别，如果将`stratus`设置为较低的级别就会看到很多关于`log4j2`本身的日志，如加载`log4j2`配置文件的路径等信息  
②`monitorInterval`：含义是每隔多少秒重新读取配置文件，可以不重启应用的情况下修改配置  
- `Appenders`：输出源，用于定义日志输出地方  
  `log4j2`支持的输出源有很多，有控制台`Console`、文件`File`、`RollingRandomAccessFile`、`MongoDB`、`Flume` 等  
  ①`Console`：控制台输出源是将日志打印到控制台上，开发的时候一般都会配置，以便调试  
  ②`File`：文件输出源，用于将日志写入到指定的文件，需要配置输入到哪个位置（例如：`D:/logs/mylog.log`）  
  ③`RollingRandomAccessFile`: 该输出源也是写入到文件，不同的是比`File`更加强大，可以指定当文件达到一定大小（如`20MB`）时，另起一个文件继续写入日志，另起一个文件就涉及到新文件的名字命名规则，因此需要配置文件命名规则
  这种方式更加实用，因为你不可能一直往一个文件中写，如果一直写，文件过大，打开就会卡死，也不便于查找日志。  
    - `fileName` 指定当前日志文件的位置和文件名称  
    - `filePattern` 指定当发生`Rolling`时，文件的转移和重命名规则  
    - `SizeBasedTriggeringPolicy` 指定当文件体积大于size指定的值时，触发Rolling  
    - `DefaultRolloverStrategy` 指定最多保存的文件个数  
    - `TimeBasedTriggeringPolicy`   这个配置需要和`filePattern`结合使用，注意`filePattern`中配置的文件重命名规则是`${FILE_NAME}-%d{yyyy-MM-dd HH-mm}-%i`，最小的时间粒度是`mm`，即分钟  
    - `TimeBasedTriggeringPolicy`指定的size是1，结合起来就是每1分钟生成一个新文件。如果改成`%d{yyyy-MM-dd HH}`，最小粒度为小时，则每一个小时生成一个文件  
    - `NoSql`：MongoDb, 输出到MongDb数据库中  
    - `Flume`：输出到`Apache Flume`（Flume是Cloudera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统，Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接受方（可定制）的能力。）  
    
    ④`Async`：异步，需要通过`AppenderRef`来指定要对哪种输出源进行异步（一般用于配置`RollingRandomAccessFile`）  
    - `PatternLayout`：控制台或文件输出源（`Console、File、RollingRandomAccessFile`）都必须包含一个`PatternLayout`节点，用于指定输出文件的格式（如 日志输出的时间 文件 方法 行数 等格式），例如 `pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"`  
    
- `Loggers`：日志器
日志器分根日志器`Root`和自定义日志器，当根据日志名字获取不到指定的日志器时就使用`Root`作为默认的日志器，自定义时需要指定每个`Logger`的名称name（对于命名可以以包名作为日志的名字，不同的包配置不同的级别等），日志级别level，相加性`additivity`（是否继承下面配置的日志器）， 对于一般的日志器（如`Console、File、RollingRandomAccessFile`）一般需要配置一个或多个输出源`AppenderRef`；
  ①每个`logger`可以指定一个`level`（`TRACE, DEBUG, INFO, WARN, ERROR, ALL or OFF`），不指定时`level`默认为`ERROR`  
  ②`additivity`指定是否同时输出`log`到父类的`appender`，缺省为`true`。
```xml
<Logger name="rollingRandomAccessFileLogger" level="trace" additivity="true">  
	<AppenderRef ref="RollingRandomAccessFile" />  
</Logger>
```
`properties`: 属性
使用来定义常量，以便在其他配置的时候引用，该配置是可选的，例如定义日志的存放位置`D:/logs`


