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