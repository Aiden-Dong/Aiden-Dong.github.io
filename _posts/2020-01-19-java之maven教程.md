---
layout:     post
title:      maven 教程 | maven 入门介绍
subtitle:   
date:       2020-01-19
author:     Aiden
header-img: img/java.jpg
catalog: true
tags:
    - Java
---

### 1.　pom 文件的信息

`pom.xml` 是　mvn 工程的核心文件，　mvn 是通过　pom 文件来维护工程。

下面是　pom 文件的主要组成。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- 模块信息　-->
    <parent>如果是子pom(模块)，则此处记录父pom的信息</parent>
    <artifactId>模块名称，用于标志这个模块</artifactId>
    <groupId>项目(工程)名称,用于标志一个工程(多个模块靠工程名组合在一起)</groupId>
    <version>项目版本</version>

    <!--定义依赖-->
    <dependencyManagement>预先定义依赖模板</dependencyManagement>
    <dependencies>声明依赖</dependencies>

    <!-- 构建过程 -->
    <build>定义项目的构建过程</build>
    <reporting>发布设置</reporting>

    <!-- 环境信息　-->
    <properties>定义变量</properties>
    <profiles>定义运行时参数，用户编译时候可以根据这个设置动态改变编译过程</profiles>
</project>
```

### 2. mvn 声明周期

一个典型的 Maven 构建生命周期是由以下几个阶段的序列组成的：

阶段 | 目标 | 处理 | 描述
-- | -- | -- | -- | --
prepare-resources	| resources:resources | 资源拷贝	 | 本阶段可以自定义需要拷贝的资源
compile	| compiler:compile | 编译	| 本阶段完成源代码编译
process-test-resources	| resources:testResources | 测试资源拷贝	 | 本阶段可以自定义需要拷贝的测试资源
test-compile	| compiler:testCompile | 编译测试	| 本阶段完成测试源代码编译
test | surefire:test | 进行测试过程　| 本阶段运行测试代码
package	| jar:jar | 打包	| 本阶段根据 pom.xml 中描述的打包配置创建 JAR / WAR 包
install	| install:install | 安装	| 本阶段在本地 / 远程仓库中安装工程包

完成的　`mvn` 构建过程 :

`maven` 有三个相互独立的生命周期，分别为 : `clean` 、 `default` 和 `site`。

- clean 生命周期的目的是清理项目
- default 生命周期负责项目构建
- site 生命周期负责建立项目站点。

#### 2.1 clean 的生命周期 :

clean生命周期负责项目清理，包含三个阶段：

```
pre-clean
clean                # 清理上一次构建生成的文件，默认是删除主目录下的target文件夹
post-clean
```

#### 2.2 default生命周期

```
validate
initialize
generate-sources
process-sources       # 处理项目主资源文件。对src/main/resources下的内容进行变量替换等工作后，复制到项目输出的主classpath目录中。
generate-resources
process-resources
compile               # 编译项目主代码。对src/main/java下的代码编译后，复制到项目输出的主classpath目录中。
process-classes
generate-test-sources
process-test-sources  # 处理测试资源文件。对src/test/resources下的内容进行变量替换等工作后，复制到项目输出的测试classpath目录中。
generate-test-resources
process-test-resources
test-compile          # 编译项目测试代码。对src/test/java下的代码编译后，复制到项目输出的测试classpath目录中。
process-test-classes
test                  # 使用单元测试框架执行单元测试，测试代码不会被打包或部署。
prepare-package
package               # 打包，包放在target目录下
pre-integration-test
integration-test
post-integration-test
verify
install               # 将包放置到maven本地仓库中，本地其他maven项目就可以使用该包了。
deploy                # 将包部署至远程仓库。
```

#### 2.3 site生命周期

```
pre-site
site                  # 生成项目站点文档，默认目录在target/site文件夹下
post-site
site-deploy           # 将生成的项目站点发布至服务器
```

#### 2.4 解释

```
mvn clean             # 进行 clean 的前两部操作
mvn package           # 进行 default 的操作到　package
mvn post-site         # 进行 site 的前三部操作

mvn clean package     # 进行　clean 的前两部, default 的操作到　package
```

### 3. mvn 依赖

`dependency` 用于定义所依赖的 `jar` 文件， 它的生效位置主要在本  `pom` 所在的模块还有其 子pom 所在模块当中。

`dependencyManagement` 可以用来声明我们将要使用的 jar 的模版， 声明是为了方便使用。

比如我们使用测试可以首先声明:

```xml
<dependencyManagement>
    <dependencies>
        <!-- https://mvnrepository.com/artifact/junit/junit -->
        <!--junit 测试工具-->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.4</version>
            <scope>test</scope>
        </dependency>
    </dependencie>
</dependencyManagement
```
当我们在这个模块或者它的子模块中想要使用这个依赖的时候，我们就可以在 `dependencies` 中添加定义。

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
</dependency>
```

这将默认使用上面声明的那个依赖配置。

> 需要注意的是，在父pom中声明或定义的环境，将影响当前模块跟其子模块。

### 4. 构建过程

mvn 的构建过程按照声明周期主要包含 :

- 资源拷贝                 (dafult-resources)
- 编译                     (default-compile)
- 测试资源拷贝             (default-testResources)
- 测试代码编译             (default-testCompile)
- 测试                    (defaule-test)
- 打包                    (defaule-jar)

这几个过程都有默认的插件来执行，对应的执行 id 如上。

#### 4.1 资源拷贝过程 :

**方法一** :

资源拷贝一般发生在最开始的时候， 当有对应多个输出目录需要拷贝的时候我们可以将 `configuration` 放到 `executions` 里面去。然后在`execution`中添加每一个输出。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>  <!--使用的拷贝资源插件-->
    <version>2.6</version>
    <executions>
        <execution>
            <id>copy-conf</id>                  <!-- 执行 id -->
            <phase>process-resources</phase>    <!-- 对应的声明周期过程 -->
            <goals>
                <goal>copy-resources</goal>     <!-- 要执行的插件目标-->
            </goals>
            <configuration>
                <!--对应的输出-->
                <outputDirectory>${project.build.directory}/conf</outputDirectory>
                <encoding>utf-8</encoding>
                <!--对应的原文件信息配置-->
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <filtering>true</filtering>
                        <includes>
                            <include>*.properties</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
        </execution>
        <execution>
            <id>copy-bash</id>
            <phase>process-resources</phase>
            <goals>
                <goal>copy-resources</goal>
            </goals>
            <configuration>
                <encoding>utf-8</encoding>
                <outputDirectory>${project.build.directory}/bin</outputDirectory>
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <filtering>true</filtering>
                        <includes>
                            <include>*.sh</include>
                            <include>*.bat</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**方法二** :

```xml
<resources>
    <resource>
        <directory>src/main/web</directory>
        <filtering>true</filtering>
        <targetPath>${project.build.directory}/dist/web/</targetPath>
    </resource>

    <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
        <targetPath>${project.build.directory}/dist/conf/</targetPath>
    </resource>

    <resource>
        <directory>bin</directory>
        <filtering>true</filtering>
        <targetPath>${project.build.directory}/dist/bin/</targetPath>
    </resource>
 </resources>
```

#### 4.2 编译过程 :

资源准备结束后进入编译过程。

```xml
<plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-compiler-plugin</artifactId>
     <version>3.2</version>
     <configuration>
         <source>1.8</source>
         <target>1.8</target>
         <encoding>UTF-8</encoding>
     </configuration>
 </plugin>
```

- 编译 scala 方式:

```xml
<plugin>
    <groupId>org.scala-tools</groupId>
    <artifactId>maven-scala-plugin</artifactId>
    <version>2.15.2</version>
    <executions>
        <execution>
            <goals>
                <goal>add-source</goal>
                <goal>compile</goal>
            </goals>
            <configuration>
                <includes>**/*.scala</includes>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 4.3 编译后进入测试。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.8</version>
    <configuration>
        ,,,,,
    </configuration>
</plugin>
```

#### 4.4 拷贝依赖

我一般习惯在编译完成后设置依赖的拷贝这时候，其实也可以放在资源拷贝的过程里一起进行。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.8</version>
    <executions>
        <execution>
            <id>dependency-copy</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <excludeArtifactIds>false</excludeArtifactIds>
                <stripVersion>false</stripVersion>
                <excludeScope>provided</excludeScope>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 4.5 打包过程

打包过程主要定义包含的文件

```xml
<!--打包过程-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <includes>
            <include>**/*.class</include>
            <include>**/*.model</include>
        </includes>
        <outputDirectory>${project.build.directory}/lib</outputDirectory>
    </configuration>
</plugin>
```

将依赖也打入一个jar 包中
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <outputFile>
                    ${project.build.directory}/${project.artifactId}-${project.version}-selfcontained.jar
                </outputFile>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 4.6 如何打压缩包

打压缩包需要使用 `maven-assembly-plugin` 工具

pom 中定义方式 :

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.6</version>
    <executions>
        <execution>
            <id>package-file</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <descriptors>
                    <!-- 指明 assembly.xml 文件位置 -->
                    <descriptor>src/main/java/assembly.xml</descriptor>
                </descriptors>
                <!--输出的文件名-->
                <finalName>${project.parent.artifactId}-${project.parent.version}</finalName>
                <tarLongFileMode>posix</tarLongFileMode>
            </configuration>
        </execution>
    </executions>
</plugin>
```

assembly.xml

```xml
<assembly>
    <id>bin</id>
    <formats>
        <format>tar.gz</format>
    </formats>
    <includeBaseDirectory>true</includeBaseDirectory>
    <fileSets>
        <fileSet>
            <directory>${project.build.directory}/dist/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>${project.build.directory}/dist/conf</directory>
            <outputDirectory>conf</outputDirectory>
            <fileMode>0644</fileMode>
        </fileSet>
        <fileSet>
            <directory>${project.build.directory}/dist/web</directory>
            <outputDirectory>web</outputDirectory>
        </fileSet>
    </fileSets>
    <dependencySets>
        <dependencySet>
            <outputDirectory>lib</outputDirectory>
        </dependencySet>
    </dependencySets>
</assembly>
```

### 5. scala 项目构建

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>scala-study</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <scala.version>2.12.8</scala.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-compiler</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-reflect</artifactId>
            <version>${scala.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <version>2.15.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```