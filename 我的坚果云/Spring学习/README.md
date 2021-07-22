### Spring容器


https://blog.csdn.net/qq_34598667/article/details/83245753

1. 从概念上讲，spring容器是spring框架的核心，是用来管理对象的。容器将创建对象，并将它们连接在一起，配置他们，并管理他们的整个生命周期。
2. 从具体化讲，在我们的项目中，哪个东西是spring容器呢？
   - 在java项目中，我们使用实现了org.springframework.context.ApplicationCon
     text接口的实现类。
   - 在web项目中，我们使用spring.xml，spring的配置文件。
3. 从代码上讲，一个spring容器就是某个实现了ApplicationContext接口的类的实例。也就是说，从代码层面，spring容器其实就是一个ApplicationContext（一个实例化对象）。

### 实例化Spring容器

spring容器提供了两种不同类型的spring容器：

Spring BeanFactory容器：它是最简单的容器，给DI提供了基本的支持；

ApplicationContext容器：它继承自BeanFactory，包括BeanFactory容器的所有功能。

实例化spring容器有两种方式：

#### 在类路径下寻找配置文件来实例化容器

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext(new String[]{"spring.xml"});

```

#### 在文件系统路径下寻找配置文件类实例化容器

```java
ApplicationContext ctx = new FileSystemXmlApplicationContext(new String[]{"spring.xml"});
```



### Spring的反射机制

反射是根据Class.forName("**cn.classes.OneClass**")去生成一个具体的实例。

比如，当我们需要根据传进来的参数，选择具体的实现类时，反射机制就能很好的解决问题。

Spring中使用Class实例化

```java
Class c = Class.forName("com.xy.Student");
Object bean = c.getInstance();
```

Spring调用getter,setter方法



我们以setter注入例子，bean.xml：

```xml
<bean id="id" class="com.xy.Student">
    <property name="stuName" value="xy" />
</bean>
```



```java
Class c = Class.forName("com.xy.Student");
Object bean = c.getInstance();

// 通过一些操作获取stuName对应的setter方法名
String setName = "set" + "stuName";
Methond method = c.getMethod(setName, String.Class);
method.invoke(bean,"xy")
```

这样就完成了最基本的注入操作。

Class还可以访问Annotation，这样就Spring使用注解的时候，可以完成注入的功能。

#### 控制反转（IOC）

https://www.cnblogs.com/xdp-gacl/p/4249939.html

#### 依赖注入（DI）

Spring容器在创建被调用者的实例时，会自动将调用者需要的对象实例注入给调用者。

这样，调用者通过 Spring容器 获得被调用者实例，这就称为依赖注入。

依赖注入主要有两种实现方式，分别是属性setter注入和构造方法注入。

比如调用者是UserService，被调用者是UserDAO。

#### Spring中的BeanFactory实现

https://blog.csdn.net/fanbaodan/article/details/90346043