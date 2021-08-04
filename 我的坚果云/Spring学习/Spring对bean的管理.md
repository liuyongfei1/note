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

#### 静态工厂方法实例化

#### 实例工厂方法实例化

详见：https://blog.csdn.net/qq_34598667/article/details/83246492