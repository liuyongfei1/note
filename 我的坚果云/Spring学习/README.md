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

Spring调用getter，setter方法



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

控制反转，是Spring的核心，贯穿始终。所谓Ioc，对与Spring框架来说，就是由Spring来负责控制对象的生命周期和对象间的关系。

ioc是如何做的呢？有点像通过婚介找女朋友一样，在我和女朋友之间引入一个第三者：婚姻介绍所。

将我的要求告诉婚姻介绍所，我想要找一个什么样的女朋友，比如长得像李嘉欣，身材像林熙蕾，唱歌像周杰伦等，然后婚姻介绍所就会按照我们的要求，提供一个mm，我们只需要和她谈恋爱就行。

简单明了，如果婚介给我们介绍的对象不符合要求，就直接抛出异常。整个过程不再由我们自己控制，而是有婚介这样一个类似容器的机构控制。

Spring所倡导的开发方式就是如此，所有的类都会在Spring容器中登记，告诉Spring你是个什么东西，你需要什么东西，然后Spring会在系统运行到适当的时候，把你要的东西主动给你，同时也会把你主动交给其他需要你的。

所有类的创建，销毁都是由Spring来控制，也就是说控制对象生命周期的不再是引用它的对象，而是spring。

对于某个具体的对象而言，以前是它控制其他对象，现在是所有对象都被spring控制，所以这就叫控制反转。

#### 依赖注入（DI）

1. IOC的一个重点是在系统运行中，动态的向某个对象提供它所需要的其他对象，这一点是通过DI来实现的。

2. 比如对象A需要操作数据库，以前我们总是要A中自己编写代码来获得一个Conection对象，有了Spring我们只需告诉Spring，A中需要一个Connection，至于这个Connection是如何构造，何时构造，A不需要知道。
3. 在系统运行时，spring会在适当的时候制造一个Connection，然后像打针一样，注入到A对象中，这样就完成了各个对象之间关系的控制。
   A需要依赖Connection才能正常运行，而这个Connection是由spring注入到A中去，依赖注入的名字就是这么来的。

那么DI是如何实现的呢？Java 1.3之后，一个重要的特征就是发射（reflection），它允许程序在运行的时候动态的生成对象、执行对象的方法、改变对象的属性。spring就是通过反射来实现注入的。



Spring容器在创建被调用者的实例时，会自动将调用者需要的对象实例注入给调用者。

这样，调用者通过 Spring容器 获得被调用者实例，这就称为依赖注入。

依赖注入主要有两种实现方式，分别是属性setter注入和构造方法注入。

比如调用者是UserService，被调用者是UserDAO。

#### Spring中的BeanFactory实现

https://blog.csdn.net/fanbaodan/article/details/90346043

### BeanFactory和ApplicationContext

https://blog.csdn.net/yangshangwei/article/details/74937778

Spring通过一个配置文件，描述bean与bean之间的依赖关系，利用java反射功能实例化bean，并建立bean之间的依赖关系。

Spring IOC容器在完成这些底层工作的基础上，还提供了bean实例缓存，声明周期管理，资源装载等高级服务。

BeanFactory是spring框架最核心的接口，它提供了高级IOC的配置机制。

ApplicationContext建立在BeanFactory基础上，提供了更多面向应用的功能，它提供了国际化支持和框架事件体系。

我们一般称BeanFactory为ICO容器，称ApplicationContext为应用上下文，但有时候为了行为方便，我们也将ApplicationContext称为spring容器。

对于BeanFactory和ApplicationContext的用途：

- BeanFactory是Spring框架的基础，面向Spring框架本身；
- ApplicationContext面向使用框架的开发者，几乎所有的应用场合都可以直接使用ApplicationContext而非底层的BeanFactory。

### Spring对bean的管理：三种实例化bean的方式

spring容器提供了三种对bean的实例化方式：

1. 构造器实例化；
2. 静态工厂方法实例化；
3. 实例工厂方法实例化。

#### 构造方法实例化

先建一个Demo实体类：

```java
public class Demo {
	private String name;
	// getter和setter方法略
}
```

在配置文件中使用构造方法实例化：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">
	<!-- 构造器实例化 -->
	<bean id="demo1" class="com.oak.junit.day01.Demo"></bean>
</beans>
```

测试：

```
@Test
publi void testCtx {
	// 实例化spring容器
	ApplicationContext ctx = new ClassPathXmlApplicationContext("spring.xml");
	// 取出Demo1
	Demo demo1 = ctx.getBean("demo1",Demo.class);
	Sysetem.out.println(demo1);
}
```



