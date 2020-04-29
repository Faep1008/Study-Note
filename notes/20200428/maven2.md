# Maven知识点（全）

## 一、Maven的作用
目的是一个可以用于构建和管理任何基于Java的项目的工具，想要一种标准的方式来构建项目，清晰地定义项目的组成，一种简单的方式来发布项目信息，以及一种在多个项目中共享JAR的方式。  
Maven是基于项目对象模型(POM project object model)来构建项目，用通俗点的话说就是对要构建的项目进行建模，将要构建的项目看成是一个对象（Object），用PO来指代这个对象，pom.xml就是PO对象的XML描述。  


## 二、Maven安装配置
安装后可以通过Maven的命令使用相关的功能。  
和配置JDK类似，需要配置环境变量。  

## 三、安装目录
以windows操作系统安装目录为例  
--bin 执行脚本所在的目录，如`mvn`  
--boot 里面就一个jar包：`plexus-classworlds-2.5.2.jar`，`plexus-classworlds`是一个类加载器框架，相对于默认的 java 类加载器，它提供了更丰富的语法以方便配置，Maven使用该框架加载自己的类库。  
--conf 包含`settings.xml`文件，可以全局定制maven行为  
--lib 该目录包含了maven运行时需要的java类库  

## 四、超级POM
在安装目录的`lib`包下面有个`maven-model-builder-3.2.5.jar`包，
用解压工具打开`org`->`apache`->`maven`->`model`，会在其目录下面发现一个`pom-4.0.0.xml`。  
内容如下：  

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- START SNIPPET: superpom -->
<project>
  <modelVersion>4.0.0</modelVersion>

<!-- 配置默认的远程仓库，即中央仓库 -->
  <repositories>
    <repository>
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>

<!-- 配置默认的插件远程仓库，也是用的中央仓库 -->
  <pluginRepositories>
    <pluginRepository>
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
      <releases>
        <updatePolicy>never</updatePolicy>
      </releases>
    </pluginRepository>
  </pluginRepositories>

<!-- 配工程构建信息 -->
  <build>
    <directory>${project.basedir}/target</directory>
    <outputDirectory>${project.build.directory}/classes</outputDirectory>
    <finalName>${project.artifactId}-${project.version}</finalName>
    <testOutputDirectory>${project.build.directory}/test-classes</testOutputDirectory>
    <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
    <scriptSourceDirectory>${project.basedir}/src/main/scripts</scriptSourceDirectory>
    <testSourceDirectory>${project.basedir}/src/test/java</testSourceDirectory>
    <resources>
      <resource>
        <directory>${project.basedir}/src/main/resources</directory>
      </resource>
    </resources>
    <testResources>
      <testResource>
        <directory>${project.basedir}/src/test/resources</directory>
      </testResource>
    </testResources>
    <pluginManagement>
      <!-- NOTE: These plugins will be removed from future versions of the super POM -->
      <!-- They are kept for the moment as they are very unlikely to conflict with lifecycle mappings (MNG-4453) -->
      <plugins>
        <plugin>
          <artifactId>maven-antrun-plugin</artifactId>
          <version>1.3</version>
        </plugin>
        <plugin>
          <artifactId>maven-assembly-plugin</artifactId>
          <version>2.2-beta-5</version>
        </plugin>
        <plugin>
          <artifactId>maven-dependency-plugin</artifactId>
          <version>2.8</version>
        </plugin>
        <plugin>
          <artifactId>maven-release-plugin</artifactId>
          <version>2.3.2</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>

  <reporting>
    <outputDirectory>${project.build.directory}/site</outputDirectory>
  </reporting>

  <profiles>
    <!-- NOTE: The release profile will be removed from future versions of the super POM -->
    <profile>
      <id>release-profile</id>

      <activation>
        <property>
          <name>performRelease</name>
          <value>true</value>
        </property>
      </activation>

      <build>
        <plugins>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-source-plugin</artifactId>
            <executions>
              <execution>
                <id>attach-sources</id>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-javadoc-plugin</artifactId>
            <executions>
              <execution>
                <id>attach-javadocs</id>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-deploy-plugin</artifactId>
            <configuration>
              <updateReleaseInfo>true</updateReleaseInfo>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>

</project>
<!-- END SNIPPET: superpom -->

```
超级POM是所有maven项目的父pom，所有项目都继承这个超级pom。子pom.xml会完全继承父pom.xml中所有的元素，而且对于相同的元素，一般子pom.xml中的会覆盖父pom.xml中的元素，但是有特殊的元素它们会进行合并而不是覆盖；如plugin插件。  

## 五、setting.xml配置文件
settings.xml中包含类似本地仓储位置、修改远程仓储服务器、认证信息等配置。  
settings.xml文件一般存在于两个位置：  
全局配置: `${M2_HOME}/conf/settings.xml`  
用户配置: `user.home/.m2/settings.xml`    
note：用户配置优先于全局配置。  
示例：  
```xml
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
 <!-- 配置本地仓库位置 -->
 <localRepository>D:/.m2/repository</localRepository>

  <servers>
	<!--服务器认证信息-->
	<server>
      <id>epoint-nexus</id>
      <username>epointyanfa</username>
      <password>epoint_yanfa</password>
    </server>
  </servers> 

 
  <mirrors>
<!-- 配置镜像 -->
 	<mirror>
	  <id>epoint-nexus</id>
	      <name>Epoint Nexus</name>
	      <url>http://192.168.0.99:8081/nexus/content/groups/public/</url>
	  <mirrorOf>central</mirrorOf>
	</mirror>
  </mirrors>

  <profiles>
    
    <!-- 配置私服 -->
	<profile>
	   <id>epoint-nexus</id>
	   <repositories>
        <repository>
          <id>epoint-nexus</id>
          <name>Epoint Nexus Repository</name>
          <url>http://192.168.0.99:8081/nexus/content/groups/public/</url>
		  <releases>
		    <enabled>true</enabled>
		  </releases>
		  <snapshots>
		    <enabled>true</enabled>
		  </snapshots>
        </repository>
      </repositories>
	  <pluginRepositories>
        <pluginRepository>
          <id>epoint-nexus</id>
          <name>Epoint Nexus Repository</name>
          <url>http://192.168.0.99:8081/nexus/content/groups/public/</url>
		  <releases>
		    <enabled>true</enabled>
		  </releases>
		  <snapshots>
		    <enabled>true</enabled>
		  </snapshots>
        </pluginRepository>
      </pluginRepositories>
	</profile>
	 </profiles>

<!-- 激活私服 -->
  <activeProfiles>
    <activeProfile>epoint-nexus</activeProfile>
  </activeProfiles>
</settings>

```
弄清上面配置之前先要搞懂几个概念：仓库、镜像。  
## 六、Maven仓库和镜像
### 6.1、仓库
主要分为两种：本地仓库和远程仓库，maven的规则是优先在本地仓库进行寻找，如果本地仓库没有，那么便从远程仓库进行获取并下载到本地仓库。如果二者都没有，那么会报错。  

本地仓库：本地仓库是本地的缓存副本，主要起缓存作用。在maven安装目录的conf下有个settings.xml的文件，可以对本地仓库的路径进行设置

```xml
<localRepository>D:/.m2/repository</localRepository>
```
远程仓库：私服和中央仓库都属于远程仓库。  

私服：如公司搭建的基于内网可访问的maven服务器；主要是指局域网内的maven服务器。  

### 6.2、镜像
如果仓库X可以提供仓库Y存储的所有内容，那么就可以认为X是Y的一个镜像。  
任何一个可以从仓库Y获得的构件，都能够从它的镜像中获取。  

它会拦截maven对`remote repository`的相关请求，把请求里的`remote repository`地址，重定向到`mirror`里配置的地址。配置`mirror`的目的一般是出于网速考虑。 

下面的xml示例中，<mirrorOf>的值为`central`,表示该配置为中央仓库的镜像，任何对于中央仓库的请求都会转至该镜像，用户也可以使用同样的方法配置其他仓库的镜像。另外三个元素id、name、url与一般仓库配置无异，表示该镜像仓库的唯一标识符、名称以及地址。类似地，如果该镜像需要认证，也可以基于该id配置仓库认证。  

```xml
  <mirrors>
<!-- 配置镜像 -->
 	<mirror>
	  <id>epoint-nexus</id>
	      <name>Epoint Nexus</name>
	      <url>http://192.168.0.99:8081/nexus/content/groups/public/</url>
	  <mirrorOf>central</mirrorOf>
	</mirror>
  </mirrors>
```

## 七、POM文件
Maven项目的核心是pom.xml。项目对象模型(POM project object model)，定义了项目的基本信息，用于描述项目如何构建，声明项目依赖等。  
### 7.1、pom文件中主要标签的含义
#### 7.1.1、<project>标签

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
        http://maven.apache.org/xsd/maven-4.0.0.xsd">
...
</project>
```
声明当前的项目构建信息。所有的配置和依赖信息都要在此标签内。  
#### 7.1.2、项目描述信息标签

```xml
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.epoint.ztbpb</groupId>
    <artifactId>ztb-qd-parent</artifactId>
    <version>7.1.30.1</version>
    <packaging>jar</packaging>
```
- `<modelVersion>` 指定当前构建项目的模板版本，和超级POM一样。  
- `<groupId>` 指定当前构建工程的组别ID，一般为公司网址的倒叙+业务名。
- `<artifactId>` 指定当前构建工程的名称。
- `<version>` 指定当前构建工程的版本号。主要分为`release`版和`snapshot`版。
- `<packaging>` 指定当前工程的打包类型。常用的有`war`、`jar`、`pom`和`maven-plugin`等。  
其中通过`<groupId>`、`<artifactId>`和`<version>`可以在Maven服务器上确定依赖。  

#### 7.1.3、<properties>标签
用来声明变量，如在标签内可以把版本号作为变量进行声明，后面dependency中用到版本号时可以用`${变量名}`的形式代替，这样做的好处是：当版本号发生改变时，只有更新`properties`标签中的变量就行了，不用更新所有依赖的版本号。

#### 7.1.4、<dependencyManagement>标签、<parent>标签和<dependencies>标签
- `<parent>`标签：  
引用`parent`的子pom，可以继承`parent`里的依赖，优点是可以消除重复引用。  
但是可能有的子工程只需要父工程中的部分的依赖，此时`parent`就显得无力。  
  
- `<dependencyManagement>`标签和`<dependencies>`标签：  
在Maven中`<dependencyManagement>`的作用其实相当于一个对所依赖jar包进行版本管理的管理器。  
- `pom.xml`文件中，jar的版本判断的两种途径：  
（1）如果`<dependencies>`里的`dependency`自己没有声明`version`元素，那么maven就会到`<dependencyManagement>`里面去找有没有对该`artifactId`和`groupId`进行过版本声明，如果有，就继承它，如果没有就会报错，告诉你必须为`dependency`声明一个`version`。  
（2）如果`<dependencies>`中的`dependency`声明了`version`，那么无论`<dependencyManagement>`中有无对该jar的`version`声明，都以`dependency`里的`version`为准。  

- `<dependencyManagement>`标签和`<dependencies>`标签区别：  
`<dependencyManagement>`只是对版本进行管理，不会实际引入jar。  
`<dependencies>`会实际下载jar包。  


## 八、依赖
### 8.1、依赖的原则
#### 1、短路优先（就近原则）
C->B->A->X1(jar)  
C->B->X2(jar)  
C依赖B，B依赖A，A和B都包含同一个不同版本的Jar，则取B的依赖版本。（c的pom.xml中不必注明jar坐标）  

#### 2、先声明优先
如果路径相同长度相同，则谁先声明，先解析谁  
C依赖A和B,A和B都包含同一个不同版本的Jar,谁依赖在前取谁的依赖版本  

### 8.2、既然有了以上的规则，为什么项目有时还会报jar包冲突的错误？
常见原因：  
（1）传递依赖导致不同版本jar包冲突，maven采用就近原则排除了依赖路径比较远的jar，如果排除的是新版本的jar包，而调用的方法是只有新jar中才有的，这样就会报错，一般是ClassNotFound这类的错误。  
（2）不同的jar包，出现了相同的类路径，这种情况，会导jvm运行时不知道执行哪个类。
解决依赖冲突，需要根据实际需求排除无用的jar包，排出时无需指定版本号，示例：

```xml
<dependency>
    <groupId>com.epoint.frame</groupId>
    <artifactId>epoint-frame-action</artifactId>
    <exclusions>
        <exclusion>
        	<groupId></groupId>
        	<artifactId></artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
### 8.3、依赖scope
`<dependency>`中还引入了`<scope>`，它主要管理依赖的部署:  
1) `compile`：编译依赖范围，适用于所有阶段，会随着项目一起发布，依赖范围默认值【会传递依赖】  
2) `test`：测试依赖范围，测试时需要,只对测试的classpath有效，如junit。这些依赖不会被打包到最终的artifact中【不会传递依赖】  
3) `provided`：已提供依赖范围，编译和测试时需要。运行时不需要,如servlet-api,因为在运行项目的时候，由于容器已经提供，就不需要Maven重复的引入一遍【不会传递依赖】  
4) `runtime`：运行时依赖范围，测试和运行时需要。编译不需要,例如面向接口编程，JDBC驱动实现jar，写代码的时候用不到只在运行时用到，这些依赖将会被打包到最终的artifact中【会传递依赖】  
5) `system`：系统依赖范围。本地依赖，不在maven中央仓库，结合systemPath标签使用，不推荐使用system依赖【会传递依赖】  
system示例：
```xml
<dependency>
  <groupId>dingding</groupId>
  <artifactId>dingding</artifactId>
  <version>2.8</version>
  <scope>system</scope>
  <systemPath>${project.basedir}/lib/taobao-sdk-java.jar</systemPath>
</dependency>
```
6) `import`：从其它的pom文件中导入依赖设置，如：  
```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>com.epoint.frame</groupId>
			<artifactId>epoint-dependency</artifactId>
			<version>9.4.1-sp1</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

### 8.4、聚合与继承
#### 8.4.1、聚合
如果想要一次构建两个项目，而不是两个模块目录下分别执行maven指令，这时候就需要使用Maven聚合。  
使用Maven聚合的几个特点：  
- pom.xml中`packaging`值必须为`POM`，否则聚合POM会报错
- 模块使用`<modules>`标签和`<module>`标签定义  

为了方便定义，通常将聚合模块放在项目目录的最顶层，其他模块则作为聚合模块的子目录存在，可读性较高。但目录结构并非强制一定要是父子结构。  

Maven首先会解析聚合模块的POM文件，分析要构建的模块，并计算出一个反应堆构建顺序，然后根据顺序依次构建各模块。反应堆是所有模块组成的一个构建结构。

#### 8.4.2、继承
类型于面向对象设计中的继承，可以创建POM的父子结构，在父POM中声明一些配置供子POM继承，以实现“一处声明，多处使用”的目的。

继承的特点
- 父模块只是为了帮助消除配置重复，因此他本身不包含除POM之外的项目文件。
- 父模块的pom.xml的`packaging`值必须要为`POM`，和聚合一样。否则子工程会报错。
- 子模块通过<parent>标签继承父模块POM

#### 8.4.3、聚合和继承的关系
##### 区别：
- 对于聚合模块来说，他知道有哪些模块被聚合，但对于被聚合的模块来说，不知道聚合模块的存在。
- 对于继承关系的父POM来说，他不知道有哪些子模块继承了它，子模块都知道父模块是什么。

##### 相同点：
- `packaging`的值都是`POM`
- 除了POM之外都没有实际内容

为了方便，一个模块POM可以既是聚合模块POM，也是父模块POM。  


## 九、生命周期
Maven有三大生命周期：`clean`、`default`和`site`  
常用的是`clean`和`default`两个声明周期。  
### 9.1、`clean`生命周期
清理项目生产的临时文件,一般是模块下的target目录  
`pre-clean`预清洁 执行实际项目清理之前所需的流程  
`clean`清洁 删除以前构建生成的所有文件  
`post-clean`后清洁 执行完成项目清理所需的流程  
### 9.2、`default`生命周期
默认生命周期有很多，以下列出一些常用的：  
`validate` ：验证这个项目是否正确，所有必需资源是否可用  
`compile` ： 编译项目的源代码  
`test`: 运行所有单元测试例子  
`package` ：打包编译后的代码成可发包格式，例如：jar,war等  
`verify` ：做一些对包的验证操作，去检测这个包是一个合法的符合标准的包  
`install` ：将包安装到本地仓库，提供给作为其他项目使用，例如：包的本地依赖  
`deploy` ：最终的结果是部署到集成环境或者正式环境，复制这个最终版本到远程仓库并分享给其他项目或者开发者使用  

Maven生命周期的步骤会默认把前面的步骤执行一遍。  

#### 9.3、maven打包时如何跳过测试？
- （1）`idea`在`maven`里集成了`skip tests`的功能，勾选即可。
- （2）`pom`中可以跳过，在`<properties></properties>`中配置。

```xml
<properties>
    <maven.test.skip>true</maven.test.skip>
    <skipTests>true</skipTests>
</properties>
```
以上POM中的两个配置方式二选一，其中第一中不会编译源代码，第二种会编译源代码生成`.class`文件，都不会执行测试。  


## 十、Maven插件
### 10.1、插件解析机制
Maven本质上是一个插件执行框架，它的核心并不执行任何具体的构建任务，所有这些任务都交给插件来完成。  
与我们所依赖的构件一样，插件也是基于坐标保存在我们的Maven仓库当中的。在用到插件的时候会先从本地仓库查找插件，如果本地仓库没有则从远程仓库查找插件并下载到本地仓库。  
为了方便用户使用和配置插件，Maven不需要用户提供完整的插件坐标信息，就可以解析得到正确的插件。比如：在用户没有提供插件版本的情况下，Maven会自动解析插件版本。  
一般来说，中央仓库所包含的插件完全能够满足我们的需要，因此也不需要配置其他的插件仓库。  

了解插件后续必须要了解两个概念，插件目标和插件绑定。  
#### 插件目标
插件目标可以理解为一个插件里的一个功能，一个插件里可能有多个功能，这些功能都聚集在一个插件里，每个功能就是一个插件目标。

#### 插件绑定
Maven的生命周期与插件相互绑定，用以完成实际的构建任务。 具体而言，就是生命周期的阶段和插件的目标相互绑定。


#### 10.1.1、插件的默认groupId
如果是maven的官方插件，pom.xml配置的时候可以省略groupId，Maven在解析的时候会自动补齐，但为了可读性不推荐这种做法。  
#### 10.1.2、解析插件版本
如果pom.xml配置时没有提供插件的版本，Maven会自动解析插件的版本。  
另外超级POM中已经指定了核心插件的版本，所有pom都可以隐式继承。  
如果用户使用某个插件没有指定版本，且不属于核心插件范畴，Maven会遍历本地仓库和所有远程仓库，计算出`latest`和`release`的值。  
maven2采用的是`latest`，即最新快照版。  
maven3采用的是`release`，即最新发布版。  

### 10.2、插件与生命周期绑定
生命周期的阶段`phase`与插件的目标`goal`相互绑定, 用以完成实际的构建任务。一个插件可以实现多个目标（Goal）。  
如:`$ mvn compiler:compile`: 冒号前是插件前缀, 后面是该插件目标(即: `maven-compiler-plugin`的`compile`目标).
而该目标绑定了`default`生命周期的`compile`阶段:因此, 他们的绑定能够实现项目编译的目的。  
  
Maven 默认为一些核心的生命周期绑定了插件目标, 当用户通过命令调用生命周期阶段时, 对应的插件目标就会执行相应的逻辑。  

|生命周期阶段|插件目标|执行任务|
|:--|:--|:--|
|clean|maven-clean-plugin:clean|清理输出目录资源|
|compile|maven-compiler-plugin:compile|编译主代码到主输出目录|
|test|maven-surefire-plugin:test|执行测试用例|
|package|maven-jar-plugin:jar|打jar包
|install|maven-install-plugin:install|将项目输出安装到本地仓库|
|deploy|maven-deploy-plugin:deploy|将项目输出部署到远程仓库|

### 10.3、常用插件
#### 10.3.1、打包插件
`maven-jar-plugin`，打包（jar）插件，设定`MAINFEST.MF`文件的参数。  
```xml
<!-- 可以从 SVN 中获取版本号，并将其变成环境变量，交由其他插件使用 -->
<plugins>
    <plugin>
        <groupId>com.google.code.maven-svn-revision-number-plugin</groupId>
        <artifactId>svn-revision-number-maven-plugin</artifactId>
        <version>1.13</version>
        <executions>
            <execution>
                <phase>validate</phase>
                <goals>
                    <goal>revision</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <entries>
                <entry>
                    <prefix>svn</prefix>
                </entry>
            </entries>
        </configuration>
        <dependencies>
            <dependency>
                <groupId>org.tmatesoft.svnkit</groupId>
                <artifactId>svnkit</artifactId>
                <version>1.9.3</version>
            </dependency>
        </dependencies>
    </plugin>
    <!-- 构建生成时间戳 -->
    <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>buildnumber-maven-plugin</artifactId>
        <version>1.4</version>
        <configuration>
            <timestampFormat>yyyy-MM-dd HH:mm:ss.S</timestampFormat>
            <timestampPropertyName>buildTime</timestampPropertyName>
        </configuration>
        <executions>
            <execution>
                <phase>validate</phase>
                <goals>
                    <goal>create-timestamp</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    <!--打包（jar）插件，设定 MAINFEST.MF文件的参数-->
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>${maven-jar-plugin-version}</version>
        <configuration>
            <archive>
                <compress>true</compress>
                <addMavenDescriptor>false</addMavenDescriptor>
                <manifest>
                    <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
                    <addDefaultSpecificationEntries>true</addDefaultSpecificationEntries>
                </manifest>
                <manifestEntries>
                    <SVN-Revision>${svn.revision}</SVN-Revision>
                    <SVN-CommittedRevision>${svn.committedRevision}</SVN-CommittedRevision>
                    <Build-Time>${buildTime}</Build-Time>
                </manifestEntries>
            </archive>
            <excludes>
                <exclude>**/allatori.xml</exclude>
                <exclude>**/rebel.xml</exclude>
            </excludes>
        </configuration>
    </plugin>
</plugins>
```

打包后的示例：  
```
Manifest-Version: 1.0
Implementation-Title: BidStandardPBCore
Implementation-Version: 7.1.30.1
Archiver-Version: Plexus Archiver
Built-By: Faep
Specification-Title: BidStandardPBCore
Implementation-Vendor-Id: com.epoint.ztbpb
SVN-CommittedRevision: 15827
Build-Time: 2020-04-27 17:29:10.199
Created-By: Apache Maven 3.2.5
Build-Jdk: 1.8.0_60
Specification-Version: 7.1.30.1
SVN-Revision: 15828
```

属性解释：  
- Manifest-Version：用来定义manifest文件的版本
- Implementation-Title：定义了扩展实现的标题
- Implementation-Version：定义扩展实现的版本
- Archiver-Version：存档版本
- Built-By：构建者（打包者）
- Specification-Title：定义扩展规范的标题
- Implementation-Vendor-Id：定义扩展实现的组织的标识
- SVN-CommittedRevision：SVN提交版本
- Build-Time：构建时间（打包时间）
- Created-By：声明该文件的生成者，一般该属性是由jar命令行工具生成的
- Build-Jdk：打包使用的JDK版本
- Specification-Version：定义扩展规范的版本
- SVN-Revision：本地SVN版本

#### 10.3.2、Tomcat插件
在war工程的pom中  
```xml
<build>
    <plugins>
    <!-- tomcat7运行插件 -->
		<plugin>
			<groupId>org.apache.tomcat.maven</groupId>
			<artifactId>tomcat7-maven-plugin</artifactId>
			<version>${tomcat7_maven_plugin_version}</version>
			<configuration>
				<path>/${project.artifactId}</path>
				<port>8087</port>
				<uriEncoding>${file_encoding}</uriEncoding>
				<url>http://localhost:8087/</url>
				<server>tomcat7</server>
			</configuration>
		</plugin>
	</plugins>
</build>
```
运行插启动tomcat，执行命令`mvn -tomcat7:run`

#### 10.3.3、编译插件
指定编译版本
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<version>${maven_compiler_plugin_version}</version>
	<configuration>
		<source>${java_version}</source><!-- 源代码使用的开发版本 -->
		<target>${java_version}</target><!-- 需要生成的目标class文件的编译版本 -->
		<encoding>${file_encoding}</encoding>
	</configuration>
</plugin>
```

### 10.4、个性化插件
#### 10.4.1、主要步骤说明
- 创建一个`maven-plugin`项目，插件本身也是`maven`项目，区别就是`packaging`标签是`maven-plugin`。可以用`maven-archetype-plugin`快速创建。  
- 为插件写目标，每个插件都必须包含一个或多个目标，Maven称之为`Mojo`（`Maven Old Java Object`），和Java的POJO对应。编写插件的时候必须提供一个或多个继承自`AbstractMojo`的类。
- 为目标提供配置点，大部分的Maven插件及其目标都是可配的。
- 编写代码实现目标行为，根据实际的需要实现Mojo。
- 错误处理及日志，Mojo发生异常时，应该提供可靠的日志信息。
- 插件测试

#### 10.4.2、案例
编写一个用于代码统计的案例：使用该插件，用户可以了解到Maven项目中各个源代码目录下文件的数量，以及它们加起来共有多少行代码  
  
首先创建一个`maven-plugin`项目，确保其pom中有如下内容：  

```xml
<properties>
	<maven.version>3.0</maven.version>
</properties>
<dependencies>
	<dependency>
		<groupId>org.apache.maven</groupId>
		<artifactId>maven-plugin-api</artifactId>
		<version>${maven.version}</version>
	</dependency>
	<dependency>
		<groupId>org.apache.maven.plugin-tools</groupId>
		<artifactId>maven-plugin-annotations</artifactId>
		<version>3.2</version>
		<scope>provided</scope>
	</dependency>
	<dependency>
		<groupId>org.codehaus.plexus</groupId>
		<artifactId>plexus-utils</artifactId>
		<version>3.0.8</version>
	</dependency>
</dependencies>
```
然后写一个Mojo类继承抽象类`AbstractMojo`：
```java
package com.epoint.maven.plugin.CodeCountPlugin;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import org.apache.maven.model.Resource;
import org.apache.maven.plugin.AbstractMojo;
import org.apache.maven.plugin.MojoExecutionException;

/**
 * 计算代码行数
 *
 * @goal codecount
 */
public class MyMojo extends AbstractMojo {

	private static final String[] INCLUDES_DEFAULT = { "java", "xml", "properties" };

	/**
	 * @parameter expression= "${project.basedir}"
	 * @required
	 * @readonly
	 */
	private File basedir;

	/**
	 * @parameter expression= "${project.build.sourceDirectory}"
	 * @required
	 * @readonly
	 */
	private File sourceDirectory;

	/**
	 * @parameter expression= "${project.build.testSourceDirectory}"
	 * @required 
	 * @readonly
	 */
	private File testSourceDirectory;

	/**
	 * @parameter expression= "${project.build.resources}" 
	 * @required 
	 * @readonly
	 */
	private List<Resource> resources;

	/**
	 * @parameter expression= "${project.build.testResources}" 
	 * @required 
	 * @readonly 
	 */
	private List<Resource> testResources;

	/**
	 * 将包括在内的文件类型进行计数
	 *
	 * @parameter
	 */
	private String[] includes;

	public void execute() throws MojoExecutionException {
		if (includes == null || includes.length == 0) {
			includes = INCLUDES_DEFAULT;
		}
		try

		{
			countDir(sourceDirectory);
			countDir(testSourceDirectory);
			for (Resource resource : resources) {
				countDir(new File(resource.getDirectory()));
			}
			for (Resource resource : testResources) {
				countDir(new File(resource.getDirectory()));
			}
		} catch (IOException e) {
			throw new MojoExecutionException("无法计算代码行.", e);
		}

	}

	private void countDir(File dir) throws IOException {
		if (!dir.exists()) {
			return;
		}
		List<File> collected = new ArrayList<>();
		collectFiles(collected, dir);
		int lines = 0;
		for (File sourceFile : collected) {
			lines += countLine(sourceFile);
		}
		String path = dir.getAbsolutePath().substring(basedir.getAbsolutePath().length());
		getLog().info("目录：" + path + "，有 " + collected.size() + "个文件， 共计" + lines + " 行代码 ");
	}

	/**
	 * 递归收集目录下所有应该被统计的文件
	 * @param collected
	 * @param file
	 */
	private void collectFiles(List<File> collected, File file) {
		if (file.isFile()) {
			for (String include : includes) {
				if (file.getName().endsWith("." + include)) {
					collected.add(file);
					break;
				}
			}
		} else {
			for (File sub : file.listFiles()) {
				collectFiles(collected, sub);
			}
		}

	}

	/**
	 * 统计单个文件的行数
	 * @param file
	 * @return
	 * @throws IOException
	 */
	private int countLine(File file) throws IOException {
		BufferedReader reader = new BufferedReader(new FileReader(file));
		int line = 0;
		try {
			while (reader.ready()) {
				reader.readLine();
				line++;
			}
		} finally {
			reader.close();
		}
		return line;
	}
}

```
每个插件目标类，或者说Mojo，都必须继承抽象类`AbstractMojo`并实现`execute()`方法，只有这样Maven才能识别该插件目标，并执行`execute()`方法中的行为。  
提供`@goal`标注，任何一个Mojo都必须使用改标注写明自己的目标名称，有了目标定义后，才能在项目中配置该插件目标，或者在命令行调用它。  
总结：  
创建一个Mojo锁必要的三项工作：继承抽象类`AbstractMojo`，实现`execute()`方法，提供`@goal`标注。  

插件配置点  
使用`parameter`参数表示用户可以在使用该插件的时候在POM文件中配置改字段。如上述代码中的： 
```java
/**
 * 将包括在内的文件类型进行计数
 *
 * @parameter
 */
private String[] includes;
```

另外的说明：  
- `@required`：表示该Mojo参数是必须的，如果使用了该标注，用户没有配置值且没有默认值的话会报错。  
- `@readonly`：表示该Mojo参数是只读的，如果使用了该标注，用户就无法对其进行配置。
- `@parameter expression="${aSystemProperties}"`：使用系统属性表达式对Mojo属性赋值。

插件写好之后编译并使用`maven install`安装到本地仓库后就可以在其他项目中使用。  
如：
```xml
<build>
    <plugins>
		<plugin>
			<groupId>com.epoint.maven.plugin</groupId>
			<artifactId>CodeCountPlugin</artifactId>
			<version>0.0.1-SNAPSHOT</version>
			<executions>
				<execution>
					<goals>
						<goal>codecount</goal>
					</goals>
					<phase>package</phase>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```
其中：
`<goal>`表示使用的插件目标，即代码中的`@goal`标注定义的名称。  
`<phase>`表示使用插件的生命周期阶段，此处使用的是`package`打包阶段。  

针对上面所说的,使用`parameter`参数表示用户可以在使用该插件的时候在POM文件中配置改字段。示例如下：
```xml
<build>
    <plugins>
		<plugin>
			<groupId>com.epoint.maven.plugin</groupId>
			<artifactId>CodeCountPlugin</artifactId>
			<version>0.0.1-SNAPSHOT</version>
			<configuration>
				<includes>
					<include>java</include>
					<include>html</include>
				</includes>
			</configuration>
			<executions>
				<execution>
					<goals>
						<goal>codecount</goal>
					</goals>
					<phase>package</phase>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

调用插件后，打包的输出会多出如下的行：
```
[INFO] --- CodeCountPlugin:0.0.1-SNAPSHOT:codecount (default) @ EpointPB.StandradCore ---
[INFO] 目录：\src\main\java，有 148个文件， 共计31072 行代码 
[INFO] 目录：\src\test\java，有 0个文件， 共计0 行代码 
[INFO] 目录：\src\main\resources，有 0个文件， 共计0 行代码 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 12.395 s
[INFO] Finished at: 2020-04-28T14:49:15+08:00
[INFO] Final Memory: 38M/307M
[INFO] ------------------------------------------------------------------------
```

## 十一、其他知识点
### 11.1、关于发布Jar包
发布Jar包到私服其实是发布了Jar包和POM文件两个单独的内容。可通过本地仓库发现这一点。  
另外，在pom文件中`<dependency>`标签下可通过`<type>`指定引入的类型，比如`jar`或`pom`，默认是`jar`。

### 11.2、POM中的<Profile>标签
主要用于针对不同环境的构建
