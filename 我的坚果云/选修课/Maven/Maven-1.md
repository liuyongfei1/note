### Maven坐标的介绍

每个maven项目都有一个坐标

groupId + artifactId + version + packaging + classifier。

五个维度（后边两个比较少用）的坐标，唯一定位一个依赖包。



### spring+Mybatis的整合

src/main/resources目录下：

mybatis的配置文件 mybatis-config.xml

spring的配置文件 application.xml

jdbc.properties

log4j.properties

### 依赖范围

<scope></scope>

#### compile

默认，对编译、测试、运行的classpath都有效。一般都是这样scope。

#### test

仅仅在编译和运行测试代码的classpath有效，编译或者运行主代码的时候无效。

仅仅测试代码需要的依赖一般都会设置为这个范围，比如Junit。一些测试框架，或者只要在测试代码中才会使用的一些依赖，会设置为test。

这个好处在于，打包的时候这种test scope的是不会放到最终的发布包里去的，减少发布包的体积。

#### provided

编译和测试的时候有效，但是在运行的时候无效。因为可能环境已经提供了，比如servlet-api。一般这个范围，在运行的时候，servlet容器会提供依赖。

servlet-api是用来开发java web项目的，可能你在开发代码和执行单元测试的时候，需要在pom.xml中声明这个servlet-api的依赖，因为要写代码和测试代码。但是最终打完包之后，放到tomact容器里去跑的时候，是不需要将这个servlet-api的依赖包打入发布包中的，因为tomcat容器本身就会给你提个这个servlet-api包。

还比如说，一个Spring Boot项目，自带有tomcat：

```xml
 <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
 </dependency>
```

如果想把项目部署到服务器上装好的tomcat，也就是说不使用spring boot自带的tomcat，那么通常会选择将项目打成war包。需要改动三个地方。

##### 第一个地方：修改pom文件里的打包方式

修改为war：

```xml
<packaging>war</packaging>
```

##### 第二个地方：排除掉自带的tomcat

那么这时就需要把spring boot自带的tomcat给排除掉：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter-tomcat</actifactId>
	  <scope>provided</scope>
</dependency>
```

把内嵌的tomcat在发布的时候给排除掉，只在编译和测试的时候用。

##### 第三个地方：修改启动类

<img src="Maven-1.assets/image-20211212083548653.png" alt="image-20211212083548653" style="zoom:50%;" />

将前后端项目关联起来

nginx这里需要添加配置：

凡是 prod-api的请求路径，都代理到 192.168.31.101机器上的8080端口对应的服务：

<img src="Maven-1.assets/image-20211212084504178.png" alt="image-20211212084504178" style="zoom:50%;" />

如果是用tomcat部署，启动tomcat后，默认的访问是tomcat的一个默认访问地址，怎么给修改成默认打开的是我们的项目呢？

需要在101机器的tomcat的配置文件service.xml里，添加一行配置：

<img src="Maven-1.assets/image-20211212084940094.png" alt="image-20211212084940094" style="zoom:50%;" />

现在后端项目在102机器上也同时部署了一个，要和101上的服务做成了一个小集群，则需要修改nginx配置。

在101集群的tomcat配置文件service.xml里：

<img src="Maven-1.assets/image-20211212085843833.png" alt="image-20211212085843833" style="zoom:50%;" />



修改之前的代理转发的配置：

<img src="Maven-1.assets/image-20211212085605620.png" alt="image-20211212085605620" style="zoom:50%;" />

为：

<img src="Maven-1.assets/image-20211212085746859.png" alt="image-20211212085746859" style="zoom:50%;" />

#### runntime

测试和运行时classpath有效，但是编译代码时无效。比如jdbc的驱动实现类，比如mysql驱动、

因为写代码的时候基于javax.sql包下的标准接口去写代码的。然后在测试的时候需要用到这个包，在实际运行的时候也需要这个包，但是编译的时候只用javax.sql接口就可以了，不需要mysql驱动类。

但一般我们声明mysql驱动的时候不会设置为runtime，因为也许你开发代码的时候会用到mysql驱动特定的api接口，不仅仅只是用javax.sql。

### maven plugin

phase plugin

plugin绑定到phase上。

如何配置插件，插件配置的语法是什么。

插件执行的原理，能看懂插件的配置。

每个插件有多个goal，一个goal是一个功能，一个插件有多个goal。

<resources>	

​	<resource>

​	</resource>

</resources>

这段配置就决定了将要把哪些资源拷贝到要打的jar包目录里去。



如果我们要使用某个maven插件，**如何手动将插件的goal绑定到phase上呢**？



比如将source插件的jar-no-fork goal绑定到verify phase，在完成集成测试之后，就生成源码的jar包，这里可以看到绑定plugin的语法：



会跑到镜像配置的私服里去找



给 verify 这个phase 绑定了 jar-no-fork  这个 goal。

这个插件会执行一次，什么时候执行呢？到 verify 这个phase的时候会执行。

#### maven生命周期

mave有三套完全独立的生命周期，clean、default和site。每套生命周期都可以独立运行，每个生命周期的运行都包含多个phase，每个phase又是有各种插件的goal来完成的，一个插件的goal可以认为是一个功能。

每次执行一个生命周期，都会一次运行这个生命周期内部的多个phase，每个phase执行时都会执行某个插件的goal完成的具体功能。

clean生命周期包含的phase如下：

 

pre-clean

clean

post-clean

 

default生命周期包含的phase如下：

 

validate：校验这个项目的一些配置信息是否正确

initialize：初始化构建状态，比如设置一些属性，或者创建一些目录

generate-sources：自动生成一些源代码，然后包含在项目代码中一起编译

process-sources：处理源代码，比如做一些占位符的替换

generate-resources：生成资源文件，才是干的时我说的那些事情，主要是去处理各种xml、properties那种配置文件，去做一些配置文件里面占位符的替换

process-resources：将资源文件拷贝到目标目录中，方便后面打包

compile：编译项目的源代码

process-classes：处理编译后的代码文件，比如对java class进行字节码增强

generate-test-sources：自动化生成测试代码

process-test-sources：处理测试代码，比如过滤一些占位符

generate-test-resources：生成测试用的资源文件

process-test-resources：拷贝测试用的资源文件到目标目录中

test-compile：编译测试代码

process-test-classes：对编译后的测试代码进行处理，比如进行字节码增强

test：使用单元测试框架运行测试

prepare-package：在打包之前进行准备工作，比如处理package的版本号

package：将代码进行打包，比如jar包

pre-integration-test：在集成测试之前进行准备工作，比如建立好需要的环境

integration-test：将package部署到一个环境中以运行集成测试

post-integration-test：在集成测试之后执行一些操作，比如清理测试环境

verify：对package进行一些检查来确保质量过关

install：将package安装到本地仓库中，这样开发人员自己在本地就可以使用了

deploy：将package上传到远程仓库中，这样公司内其他开发人员也可以使用了

 

site生命周期的phase：

 

pre-site

site

post-site

site-deploy

####  maven的命令行和声明周期

比如 maven clean package

clean 是指 default 生命周期中的 clean phase，

package 是指 package 生命周期中的 package phase。

此时就会执行clean 生命周期中，在clean phase 之前的所有 phase和 clean phase，

然后会执行 default 生命周期中，在  package phase之前的所有 phase和  package phase。

### Maven运行单元测试及输出覆盖率报告

maven中默认内置了surefire插件来运行单元测试，与最新流行的junit单元测试框架整合非常好。

一般是在default生命周期的test阶段，会运行surefire插件的test  goal，然后执行 src/test/java 下面的所有单元测试。默认，如果单元测试失败的话，就会在src/test/java/surefire-reports目录下生成错误报告。

同时可以引入 cobertura 插件，可以看到测试覆盖率的报告。

### Maven工程中各模块依赖版本统一

在父工程中，使用<dependencyManagement> 和 <pluginManagement>来声明所有的依赖和插件。

此时在子工程中，就可以对自己需要的依赖进行声明，而不用写版本号。只有在子工程中声明了，才会继承依赖，而且版本由父工程约束。

**如果在父工程中，直接用dependencies和plugins来声明依赖和插件，子工程会强制全部继承；**

**如果用dependencyManagemnent和pluginManagement来声明依赖和插件，默认情况下，子工程什么都不继承，只有当子工程声明了某个依赖或者插件的groupId+artifactId，但是不指定版本时，才会从父工程继承那个依赖。**

### Maven自动化部署

基于maven做自动化部署。在maven中部署cargo插件，会将war包部署到比如tomcat的webapps目录。