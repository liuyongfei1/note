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

#### test

仅仅在编译和运行测试代码的classpath有效，编译或者运行主代码的时候无效。

仅仅测试代码需要的依赖一般都会设置为这个范围，比如Junit。一些测试框架，或者只要在测试代码中才会使用的一些依赖，会设置为test。

这个好处在于，打包的时候这种test scope的是不会放到最终的发布包里去的，减少发布包的体积。

#### provided

编译和测试的时候有效，但是在运行的时候无效。因为可能环境已经提供了，比如servlet-api。一般这个范围，在运行的时候，servlet容器会提供依赖。

servlet-api是用来开发java web项目的，可能你在开发代码和执行单元测试的时候，需要在pom.xml中声明这个servlet-api的依赖，因为要写代码和测试代码。但是最终打完包之后，放到tomact容器里去跑的时候，是不需要将这个servlet-api的依赖包打入发布包中的，因为tomcat容器本身就会给你提个这个servlet-api包。

#### runntime

测试和运行是classpath有效，但是编译代码时无效。