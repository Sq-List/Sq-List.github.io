---
title: Maven笔记
date: 2021-02-16 17:31:27
tags:
	- Maven
categories:
	- Maven
---

内容参照：[Link](http://c.biancheng.net/maven/)

### 标准目录结构

```
<project>
+- src
|  +- main
|     +- java
|     +- resource
|     \- webapp (web项目才需要有) 
|        \- web.xml
|  +- test
|     +- java
|     \- resource
\- pom.xml
```



<!-- more -->



### 常用命令

命令的本质是执行 maven plugin

1. mvn clean: 清除
   清除target目录
2. mvn compile: 编译
   将项目中的.java文件编译成.class文件
3. mvn test: 单元测试
   将项目根目录下的src/test/java目录下的单元测试类执行（文件名必须以 Test.java 结尾）
4. mvn package: 打包
   生成 jar/war 由项目决定
5. mvn install: 安装
   本地多个项目公用一个jar包，打包到本地仓库



### 生命周期（[link](http://c.biancheng.net/view/4899.html)）

Maven 中项目的构建生命周期只是 Maven 根据实际情况**抽象提炼出来的一个统一标准和规范**，是不能做具体事情的。也就是说，Maven 没有提供一个编译器能在编译阶段编译源代码。具体事情，由集成到 Maven 中的**插件完成**。对大多数构建阶段绑定了默认的插件，通过这样的默认绑定，又简化和稳定了实际项目的构建。

maven中粗在“三套”生命周期，每一套生命周期相互独立，互不影响。它们分别是 clean、default 和 site。

* clean：生命周期的目的是**清理项目**；
* default：生命周期的目的是**构建项目**；
* site：生命周期的目的是**建立项目站点**。

在一套生命周期内，之后后面的命令前面的操作会自动执行。

#### clean 生命周期

clean生命周期的目的是清理项目，包括以下三个阶段。

* pre-clean：执行清理前需要完成的工作。
* clean：清理上一次构建过程中的文件，比如编译后的class文件等。
* post-clean：执行清理后需要完成的工作

#### default 生命周期

| 名称                        | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| validate                    | 验证项目结构是否正常，必要的配置文件是否存在                 |
| initialize                  | 做构建前的初始化操作，比如初始化参数、创建必要的目录等       |
| generate-sources            | 产生在编译过程中需要的源代码                                 |
| process-sources             | 处理源代码，比如过滤值                                       |
| **generate-resources**      | 产生主代码中的资源在classpath中的包                          |
| **process-resources**       | 将资源文件复制到classpath的对应包中                          |
| **compile**                 | 编译项目中的源代码                                           |
| precess-classes             | 产生编译过程中生成的文件                                     |
| generate-test-sources       | 产生编译过程中测试相关的代码                                 |
| process-test-sourccompilees | 处理测试代码                                                 |
| **generate-test-resources** | 产生测试中资源在classpath中的包                              |
| **process-test-resources**  | 将测试资源复制到classpath中                                  |
| **test-compile**            | 编译测试代码                                                 |
| process-test-classes        | 产生编译测试代码过程的文件                                   |
| **test**                    | 运行测试案例                                                 |
| prepare-package             | 处理打包前需要初始化的准备工作                               |
| package                     | 将编译后的class和资源打包成压缩文件，比如rar                 |
| pre-integration-test        | 做好集成测试前的准备工作，比如集成环境的参数设置             |
| integration-test            | 集成测试                                                     |
| post-integration-test       | 完成集成测试后的收尾工作，比如清理集成环境的值               |
| verify                      | 检测测试后的包是否完好                                       |
| **install**                 | 将打包的组件以构件的形式，安装到本地依赖仓库中，以便共享给本地的其他项目 |
| **deploy**                  | 运行集成和发布环境，将测试后的最终包以构件的方式发布到远程仓库中，方便所有程序员共享 |

简单流程：compile -> test -> package -> install -> deploy

#### site 生命周期

site 生命周期的目的是建立和发布项目站点。Maven 可以基于 pom 所描述的信息自动生成项目的站点，同时还可以根据需要生成相关的报告文档集成在站点中，方便团队交流和发布项目信息。site 生命周期包括如下阶段。

- pre-site：执行生成站点前的准备工作。
- site：生成站点文档。
- post-site：执行生成站点后需要收尾的工作。
- site-deploy：将生成的站点发布到服务器上。



### 插件的获取和配置

Maven只是对项目的构建过程进行了统一的抽象定义和管理。至于每个阶段由谁来做，Maven 自己不去实现，而是让对应的**插件去完成**。

一个插件可以**实现生命周期多个阶段的任务**，比如 maven-dependency-plugin 就可以实现十多个功能：分析项目的依赖功能；列出项目的依赖树；分析依赖的来源等。

将插件的每个功能叫目标，这样就可以实现在哪个阶段，执行哪个插件，达到哪个目标。比如：“dependency:tree”表示 maven-dependency-plugin 列出依赖的目标。

#### 插件同生命周期的阶段的绑定

##### 内置绑定

Maven 自动为生命周期的主要阶段绑定很多插件的目标。

##### 自定义绑定

```xml
<project>
    ...
    <build>
        <plugins>
            ...
            <plugin>
                <!-- 指定插件 -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.0.0</version>
                <executions>
                    <execution>
                        <id>att-sources</id>
                        <!-- 配置绑定的阶段，不指定会使用插件目标默认的绑定的阶段 -->
                        <phase>verify</phase>
                        <goals>
                            <!-- 指定插件的目标 -->
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    ...
</project>
```



### 插件参数配置

#### 命令行配置参数

使用 -D后面接参数名称＝参数值的方式配置目标参数。

maven-surefire-plugin 插件中提供了一个 maven.test.skip 参数，当它的值为 true 时，就不会执行 test 案例。

```shell
mvn install -Dmaven.test.skip=true
```

#### pom配置参数

有些参数在项目创建好之后，目标每次执行的时候都不需要改变，这是可以将这些值配置到pom.xml文件中。

```shell
mvn help:describe -Dplugin=org.apache.maven.plugins:maven-compiler-plugin:3.5.1 -Ddetail 
```

会发现在compile目标中有一堆参数，其中：

```
source (Default: 1.5)
User property: maven.compiler.source
The -source argument for the Java compiler.

staleMillis (Default: 0)
User property: lastModGranularityMs
Sets the granularity in milliseconds of the last modification date for
testing whether a source needs recompilation.

target (Default: 1.5)
User property: maven.compiler.target
The -target argument for the Java compiler.
```

有source和target两个参数的介绍，可以通过pom.xml做如下配置，

```xml
<project>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
                <configuration>
                    <!-- 指定编译 Java 1.5 的源代码 -->
                    <source>1.5</source>
                    <!-- 生成于 JVM 1.5 兼容的字节码文件 -->
                    <target>1.5</target>
                </configuration>
            </plugin>
            ...
        </plugins>
    </build>
    ...
</project>
```

这种配置是给maven-compiler-plugin插件配置一个全局的参数值，也可以给特定的任务指定特定的值：

```xml
<project>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <id>att-sources</id>
                        <configuration>
                            <!-- 指定编译 Java 1.5 的源代码 -->
                            <source>1.5</source>
                            <!-- 生成于 JVM 1.5 兼容的字节码文件 -->
                            <target>1.5</target>
                        </configuration>
                    </execution>
              	</executions>
            </plugin>
            ...
        </plugins>
    </build>
    ...
</project>
```

这样这两个值就只对某一个任务生效。



###调用插件

```shell
mvn <插件名称|前缀>:<目标> [-D参数名=参数值 ...]
```

如

```shell
mvn dependency:tree
```



### 解析插件

上面的

```shell
mvn dependency:tree
```

这里只用 ```dependency```就指定了 maven-dependency-plugin 插件。

maven为方便用户使用，可以用插件的前缀来指定插件。插件前缀是与<groupId>:<artifactId>是一一对应的。这种对应关系保存在仓库的元数据中，这样的元数据为<groupId>/maven-metadata.xml。

目前绝大多数的插件是放在  [http://repo1.maven.org](http://repo1.maven.org/) 和 [http://repository.codehaus.org](http://repository.codehaus.org/) 中的，他们的groupId对应的分别是 org.apache.maven.plugins 和 org.-codehaus.mojo。也就是说Maven在解析插件仓库元数据时，会默认使用org.apache.maven.plugins 和 org.codehaus.mojo 两个groupId，即[http://repo1.maven.org/maven2/org/apache/maven/plugins/maven-metadata.xml](http://repository.codehaus.org/org/codehaus/mojo/maven-metadata.xml)和 http://repository.codehaus.org/org/codehaus/mojo/maven-metadata.xml 中的元数据。

其中 org/apache/maven/plugins/maven-metadata.xml 部分内容为 

```xml
<metadata>
    <plugins>
        <plugin>
            <name>Maven Clean Plugin</name>
            <prefix>clean</prefix>
            <artifactId>maven-clean-plugin</artifactId>
        </plugin>
        <plugin>
            <name>Maven Compiler Plugin</name>
            <prefix>compiler</prefix>
            <artifactId>maven-compiler-plugin</artifactId>
        </plugin>
        <plugin>
            <name>Maven Dependency Plugin</name>
            <prefix>dependency</prefix>
            <artifactId>maven-dependency-plugin</artifactId>
        </plugin>
        ...
    </plugins>
</metadata>
```

从上面的内容中可以发现，maven-clean-plugin 的前缀是 clean，maven-compiler-plugin 的前缀是 compiler，maven-dependency-plugin 的前缀是 dependency。

如果在第一个 metadata.xml 中没有找到目标插件，就用同样的流程找其他的 metadata.xml，包括用户自己定义的 metadata.xml。如果所有的地方都没有找到对应的前缀，这就以报错的形式结束了。



### 坐标

#### groupId

1. Maven 项目和实际项目不一定是一一对应的。比如 SpringFramework，它对应的 Maven 项目就有很多，如 spring-core、spring-context、spring-security 等。造成这样的原因是模块的概念，所以一个实际项目经常会被划分成很多模块。

2. groupId 不应该同开发项目的公司或组织对应。原因比较好理解，一个公司和一个组织会开发很多实际项目，如果用 groupId 对应公司和组织，那 artifactId 就只能是对应于每个实际项目了，而再往下的模块就没法描述了，而往往项目中的每个模块是以单独的形式形成构件，以便其他项目重复聚合使用。
3. groupId 的表述形式同 Java 包名的表述方式类似，通常与域名反向一一对应。

#### artifactId

定义实际项目中的一个 Maven 项目（实际项目中的一个模块）。

推荐命名的方式为：实际项目名称-模块名称。

比如，org.springframework 是实际项目名称，而现在用的是其中的核心模块，它的 artifactId 为 spring-core。

#### version

定义 Maven 当前所处的版本。具体规范请参考《[版本管理](http://c.biancheng.net/view/4828.html)》的介绍。

#### packaging

定义 Maven 项目的打包方式。

1. 打包方式通常与所生成的构件文件的扩展名对应，比如，.jar、.ear、.war、.pom 等。
2. 打包方式是与工程构建的生命周期对应的。比如，jar 打包与 war 打包使用的命令是不相同的。
3. 默认packaging为 jar。

#### classifier

定义构件输出的附属构件。

附属构件同主构件是一一对应的，比如上面的 spring-core-4.2.7.RELEASE.jar 是 spring-core Maven spring-core 项目的主构。

Maven spring-core 项目除了可以生成上面的主构件外，也可以生成 spring-core-4.2.7.RELEASE-javadoc.java 和 spring-core-4.2.7.RELEASE-sources.jar 这样的附属构件。这时候，javadoc 和 sources 就是这两个附属构件的 classifier。这样就为主构件的每个附属构件也定义了一个唯一的坐标。

#### 区分基于不同JDK版本的jar包

```xml
<dependency>  
    <groupId>net.sf.json-lib</groupId>   
    <artifactId>json-lib</artifactId>   
    <version>2.2.2</version>  
    <classifier>jdk13</classifier>    
</dependency>  
```

#### 区分项目的不同组成部分（source、javadoc）

```xml
<dependency>  
    <groupId>net.sf.json-lib</groupId>   
    <artifactId>json-lib</artifactId>   
    <version>2.2.2</version>  
    <classifier>jdk15-javadoc</classifier>    
</dependency> 
```

#### 总结

groupId、artifactId 和 version 是必需的，packaging 是可选的，默认是 jar，而 classifier 是不能直接定义的。同时，Maven 项目的构件文件名与坐标也是有对应关系的，一般规则是 artifactId-version[-classifier].packaging。



### 仓库

Maven存放构件的仓库分两种：**本地仓库**和**远程仓库**。Maven 寻找构件的时候，先查看本地仓库，如果本地仓库存在坐标对应的构件，就直接使用。如果*本地不存在所需要的构建*，或者*需要查看是否有更新的构建版本*，Maven就会去远程仓库查找，发现需要的构建后，下载到本地仓库后使用。如果本地仓库和远程仓库都没有找到需要的构建，Maven就报错。

#### 本地仓库

Maven在根据坐标查找依赖的构件时，先是在本地仓库中查找。

一个构建只有存在本地仓库后才能被Maven项目使用。将构建保存到本地仓库最常见的有两种方式，一种是以依赖的行程，从远程仓库下载到本地；另一种是将本地项目编译打包后，安装到本地仓库。

#### 远程仓库

安装好Maven后，本地仓库是不存在的，只有执行了第一条Maven命令后，Maven才回创建本地仓库，然后根据配置和需要从远程仓库下载对应的构建到本地仓库。

本地仓库只有一个，远程仓库可以有多个。

#### 中央仓库

默认的远程仓库叫做中央仓库。

在$M2_HOME/lib/maven-model-builder-{version}.jar中，存在org/apache/maven/model/pom-4.0.0.xml文件，其中有

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

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
    ...
</project>
```

repository 配置的是依赖的默认中央仓库，pluginRepository 配置的是插件的默认中央仓库。

所有的Maven项目都会继承这个pom.xml，所以通常把这个pom叫超级pom。

#### 私服

私服是一个特殊的远程仓库，架设在局域网内。它是一个代理外网的远程仓库，供局域网内部的 Maven 用户使用。



### 解析依赖的机制

先在本地仓库中查找，如果没有找到，再找远程仓库，找到后下载到本地仓库。

如果依赖的版本为快照版本，Maven除了找到对应的构件外，还会自动查找最新的快照。过程如下：

1. 当依赖的范围是system的时候，Maven直接从本地文件系统中解析构件；
2. 根据依赖坐标计算仓库路径，尝试直接从本地仓库寻找构件，如果发现对应的构件，就解析成功；
3. 如果在本地仓库不存在相应的构件，就遍历所有的远程仓库，下载并解析使用；
4. 如果依赖的版本是RELESE或LATEST，就基于更新策略读取所有远程仓库的元数据文件（<groupId>/<artifactId>/maven-metadata.xml），将其与本地仓库的对应元合并后，计算出RELESE或LATEST真实的值，然后基于该值检查本地仓库，或者从远程仓库下载；
5. 如果依赖的版本是 SNAPSHOT，就基于更新策略读取所有远程仓库的元数据文件，将它与本地仓库对应的元数据合并，得到最新快照版本的值，然后根据该值检查本地仓库，或从远程仓库下载。
6. 如果最后解析得到的构件版本包含有时间戳，先将该文件下载下来，再将文件名中时间戳信息删除，剩下 SNAPSHOT 并使用（以非时间戳的形式使用）。



### 镜像仓库

如果仓库 A 能提供仓库 B 存储的所有服务，那么就把 A 叫作 B 的镜像。比如 http://maven.net.cn/content/groups/public 就是中央仓库 http://repo1.maven.org/maven2/ 在中国的镜像。

为了提高 [Maven](http://c.biancheng.net/maven/) 效率，可以通过配置文件用镜像代替。修改的 settings.xml 如下所示。

```xml
<settings>
  ...
  <mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  ...
</settings>
```

上面的配置中，mirrorOf 的值为 central，表示该配置为 id 为 central 仓库的镜像，也就是中央仓库的镜像。

id、name 和 url 表示镜像的唯一标记、名称和地址。如果镜像服务器需要认证，根据这个 id 配置一个对应的仓库认证。

mirrorOf的一些其他配置方式：

* <mirrorOf>*</mirrorOf>：匹配所有的远程仓库；
* <mirrorOf>external:*</mirrorOf>：匹配所有的远程仓库，使用 localhost、file:// 协议的除外；
* <mirrorOf>r1, r2</mirrorOf>：匹配指定的几个远程仓库，每个仓库之间用逗号隔开；
* <mirrorOf>*, !r1, r2</mirrorOf>：匹配除了指定仓库外的所有仓库，“!”后面的仓库是被排除外的。





### 依赖配置

依赖是配置在pom.xml文件中的，如下是关于依赖配置的大概内容：

```xml
<project>
    ...
    <dependencies>
        <dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>...</version>
            <type>...</type>
            <scope>...</scope>
            <optional>...</optional>
            <exclusions>
                <exclusion>
                  <groupId>...</groupId>
            			<artifactId>...</artifactId>
              	</exclusion>
            </exclusions>
        </dependency>
        ...
    </dependencies>
    ...
</project>
```

* groupId、artifactId 和 version：依赖的基本坐标；
* type：依赖的类型，同项目中的 packaging 对应，默认是 jar；
* scope：依赖的范围；
* optional：标记依赖是否可选；
* exclusions：排除传递性依赖。

### 依赖范围

JVM运行代码时，需要基于classpath查找需要的类文件，才能加载到内存使用。

三套classpath：

* 编译classpath；
* 测试classpath；
* 运行classpath。

默认依赖范围是 compile

| 依赖范围 | 对于编译classpath有效 | 对于测试classpath有效 | 对于运行时classpath有效 |            示例            |
| :------: | :-------------------: | :-------------------: | :---------------------: | :------------------------: |
| compile  |           Y           |           Y           |            Y            |        spring-core         |
|   test   |           -           |           Y           |            -            |           Junit            |
| provided |           Y           |           Y           |            -            |         serlet-api         |
| runtime  |           -           |           Y           |            Y            |          JDBC驱动          |
|  system  |           Y           |           Y           |            -            | 本地的，maven仓库之外的jar |

### 传递性依赖

Maven会解析项目中的每个直接依赖的pom，将那些必要的间接依赖以传递依赖的形式引入项目中。

示例，项目基于Spring框架实现时，只需将Spring的依赖配置到pom的依赖元素，至于Spring框架所依赖的第三方jar包，Maven会通过检测Spring框架的依赖信息将依赖的构件导入到项目中。

传递依赖在将间接依赖导入项目的过程中也有规则和范围，这个规则和范围是同依赖范围紧密关联的。

现有三个项目（A、B、C），假设A→B→C（→表示依赖），A对B的依赖叫第一直接依赖，B对C的依赖叫第二直接依赖，A对C的依赖叫传递依赖（通过B传递）。

A对B的第一直接依赖的范围和B对C的第二直接依赖的范围，共同决定了A到C的传递依赖范围。如下：

第一列表示第一直接依赖的范围，第一行表示第二直接依赖的范围，交叉为共同影响后的传递依赖的范围。

| 依赖     | compile  | test | provided | runtime  |
| -------- | -------- | ---- | -------- | -------- |
| compile  | compile  | -    | -        | runtime  |
| test     | test     | -    | -        | test     |
| provided | provided | -    | provided | provided |
| runtime  | runtime  | -    | -        | runtime  |

可以得到如下规律：

- 当第二直接依赖为 compile 的时候，传递依赖同第一直接依赖一致；
- 当第二直接依赖为 test 的时候，没有传递依赖；
- 当第二直接依赖为 provided 的时候，值将第一直接依赖中的 provided 以 provided 的形式传递；
- 当第二直接依赖为 runtime 的时候，传递依赖的范围基本上同第一直接依赖的范围一样，但 compile 除外，compile 的传递依赖范围为 runtime。

### 依赖的调解

当多个直接依赖都带来了同一个间接依赖，而且是不同版本的间接依赖时，就会引起重复依赖，甚至包冲突的问题。

#### 依赖调解原则

Maven依赖调解原则有：

1. 路径优先原则；
2. 声明优先原则。

路径优先原则无法解决时，再使用声明优先原则。

项目A，有两个依赖：A→B→C→T（1.0），A→D→T（2.0）。A最终对T（1.0）和T（2.0）有间接依赖。T（2.0）的路径长度为2，T（1.0）的路径长度为3，根据**最短路径原则**，将T（2.0）引入。

项目A，有两个依赖：A→B→T（1.0），A→C→T（2.0），这时两条路径都是一样的长度2，这时根据**声明优先原则**，Maven会判断哪个依赖在pom.xml中先声明，选择引入先声明的依赖。





### 排除依赖

示例：

```xml
<dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>...</version>
            <exclusions>
                <exclusion>
                  <groupId>...</groupId>
            			<artifactId>...</artifactId>
              	</exclusion>
            </exclusions>
        </dependency>
```

### 归类依赖

```xml
<project>
    ...
    <properties>
        <!-- 3.2.16.RELEASE,3.1.4.RELEASE -->
        <spring.version>4.2.7.RELEASE</spring.version>
    </properties>
    <dependencies>
        <!-- spring -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>${spring.version}</version>
        </dependency>
        ...
    </dependencies>
    ...
</project>
```

### 优化依赖

* mvn dependency:list，列出所有的依赖列表。
* mvn dependency:tree，以树形结构方式，列出依赖和层次关系。
* mvn dependency:analyze，分析主代码、测试代码编译的依赖。



### 六类属性

#### 内置属性

主要有两个内置的属性，分别是

* ${basedir}：项目的根目录，即包含pom.xml文件的目录；
* ${version}：项目的版本。

#### POM属性

引用POM文件中对应元素的值，如

* ＄{project.build.sourceDirectory}：项目的主源码目录，默认是 src/main/java。
* ＄{project.build.testSourceDirectory}：项目的测试源码目录，默认是 src/test/java。
* ＄{project.build.directory}：项目构建输出目录，默认是 target。
* ＄{project.outputDirectory}：项目主代码编译输出目录，默认是 target/classes。
* ＄{project.testOutputDirectory}：项目测试代码编译输出目录，默认是 target/testclasses。
* ＄{project.groupId}：项目的 groupId。
* ＄{project.artifactId}：项目的 artifactId。
* ＄{project.version}：项目的版本。
* ＄{project.build.finalName}：项目输出的文件名称，默认为“＄{project.artifactId}-＄{project.version}”。

#### 自定义属性

在pom的properties中定义自己的Maven属性。

#### Setting属性

用以“settings.”开头的属性引用 settings.xml 文件中 XML 元素的值。如使用“＄{settings.localRepository}”指向用户本地仓库的地址。

#### Java系统属性

所有的 Java 系统属性都可以通过 Maven 属性引用，比如“＄{user.home}”指向的就是用户目录。用户可以通过使用“mvn help:system”命令查看所有的 Java 系统属性。

#### 环境变量属性

所有的环境变量都可以用以“evn.”开头的 Maven 属性引用。比如，“＄{evn.JAVA_HOME}”就指向引用了 JAVA_HOME 环境变量的值。



### profile配置管理

为体现不同环境的不同构件，需要配置好不同环境的profile，如下：

```xml
<profiles>
    <profile>
        <id>dev_evn</id>
        <properties>
            <db.driver>com.mysql.jdbc.Driver</db.driver>
            <db.url>jdbc:mysql://localhost:3306/test</db.url>
            <db.username>root</db.username>
            <db.password>root</db.password>
        </properties>
    </profile>
    <profile>
        <id>test_evn</id>
        <properties>
            <db.driver>com.mysql.jdbc.Driver</db.driver>
            <db.url>jdbc:mysql://localhost:3306/test_db</db.url>
            <db.username>root</db.username>
            <db.password>root</db.password>
        </properties>
    </profile>
</profiles>
```

### 激活profile配置

#### 命令行激活

在mvn命令行中添加参数“-P”，指定要激活的profile的id。一次要激活多个profile，可以用逗号分开一起激活。

```bash
mvn clean install -Pdev,test
```

#### Setting文件显示激活

如希望某个profile默认一直处于激活状态，可以在**setting.xml**中配置activeProfiles元素。

```xml
<settings>
    ...
    <activeProfiles>
        <activeProfile>dev_evn</activeProfile>
    </activeProfiles>
    ...
</settings>
```

#### 系统属性激活

可以配置当某个系统属性存在时激活profile

```xml
<profiles>
    <profile>
        ...
        <activation>
            <property>
                <name>profileProperty</name>
            </property>
        </activation>
    </profile>
</profiles>
```

还可以进一步配置某个属性的值为什么时激活

```xml
<profiles>
    <profile>
        ...
        <activation>
            <property>
                <name>profileProperty</name>
                <value>dev</value>
            </property>
        </activation>
    </profile>
</profiles>
```

可以在 mvn 中用“-D”参数来指定激活

```bash
mvn clean install -DprofileProperty=dev
```

#### 操作系统环境激活

用户可以通过配置指定不同操作系统的信息，实现不同操作系统做不同的构建。

```xml
<profiles>
    <profile>
        <activation>
            <os>
                <name>Window XP</name>
                <family>Windows</family>
                <arch>x86</arch>
                <version>5.1.2600</version>
            </os>
        </activation>
    </profile>
</profiles>
```

* family为Windows、UNIX 或 Mac；
* name为操作系统名称；
* arch为操作系统的架构；
* version为操作系统的版本。

#### 文件存在与否激活

通过配置某个文件存在与否来决定是否激活profile

```xml
<profiles>
    <profile>
        <activation>
            <file>
                <missing>t1.properties</missing>
                <exists>t2.properties</exists>
            </file>
        </activation>
    </profile>
</profiles>
```

#### 默认激活

配置一个默认的激活profile

```xml
<profiles>
    <profile>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
</profiles>
```

PS：如果 pom 中有任何一个 profile 通过其他方式被激活的话，所有配置成默认激活的 profile 都会自动失效。 

可以使用如下命令查看当前激活的 profile。

```bash
mvn help:active-profiles
```

也可以使用如下命令查看所有的 profile。

```bash
mvn help:all-profiles
```

### profile的种类

#### pom.xml

pom.xml中声明的profile只对当前项目有效。

#### 用户setting.xml

在用户目录下的“.m2/settings.xml”中的 profile，对本机上的该用户的所有 Maven 项目有效。

#### 全局setting.xml

在 Maven 安装目录下 conf/settings.xml 中配置的 profile，对本机上所有项目都有效。





















