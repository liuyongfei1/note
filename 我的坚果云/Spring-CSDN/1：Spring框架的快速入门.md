### 什么是spring

https://liayun.blog.csdn.net/article/details/69663685

- spring是一个开源框架，它是为了解决企业应用开发的复杂性而创建的。

- 框架的主要优势之一就是分层架构，分层架构允许使用者选择使用哪一个组件，同时为j2ee应用程序开发提供继承的框架。

- spring的核心是控制反转（IOC）和面向切面（AOP）。简单来说，spring是一个分层的JavaSE/EE full-stack（一站式）轻量级开源框架。

为什么说spring是一个一站式的轻量级开源框架呢？EE开发可分成三次结构，针对JavaEE的三层结构，每一场spring都提供了不同的解决技术：

![为什么说spring是一个一站式的轻量级开源框架呢](Untitled.assets/为什么说spring是一个一站式的轻量级开源框架呢.png)



#### spring之beanDefinition

https://www.cnblogs.com/watertreestar/p/12830261.html

#### spring之BeanPostProcessor（后置处理器）

https://blog.csdn.net/qq_38526573/article/details/88086752

1、pojo类：

```java
/**
 * Person类
 *
 * @author Liuyongfei
 * @date 2022/1/16 21:47
 */
public class Person {

    private int id;

    private String name;

    private String beanName;

    public Person() {
        System.out.println("Person 被实例化......");
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        System.out.println("设置：" + name);
        this.name = name;
    }

    public String getBeanName() {
        return beanName;
    }

    public void setBeanName(String beanName) {
        this.beanName = beanName;
    }

    /**
     * 自定义的初始化方法
     */
    public void start() {
        System.out.println("Person中自定义的初始化方法......");
    }

}
```

2、自定义BeanPostProcessor：

```java
/**
 * 自定义的BeanPostProcessor实现类
 *
 * BeanPostProcessor接口的作用是：
 * 我们可以通过该接口中的方法，在bean实例化、配置以及其它初始化方法的前后添加一些我们自己的逻辑
 *
 * @author Liuyongfei
 * @date 2022/1/16 21:56
 */
public class MyBeanPostProcessor implements BeanPostProcessor {

    /**
     * 实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("初始化 before -- 实例化的bean对象：" + bean + "\t" + beanName);
        return bean;
    }

    /**
     * 实例化、依赖注入完毕，在调用显示的初始化完毕时执行
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("初始化 after -- 实例化的bean对象：" + bean + "\t" + beanName);
        return bean;
    }
}
```

3、applicationContext.xml：

添加：

```xml
 <bean id="person" class="com.fullstackboy.springdemo.ioc.bean.Person" init-method="start">
        <property name="name" value="张三丰"></property>
    </bean>

    <!--注册处理器-->
    <bean class="com.fullstackboy.springdemo.ioc.config.MyBeanPostProcessor"></bean>
```

4、测试类：

```java
/**
 * 测试类
 *
 * @author Liuyongfei
 * @date 2022/1/16 21:52
 */
public class BeanPostProcessorTest {

    @Test
    public void test1() {
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        Person person = context.getBean(Person.class);
        System.out.println(person);
    }
}
```

5、执行结果：

```bash
Person 被实例化......
设置：张三丰
初始化 before -- 实例化的bean对象：com.fullstackboy.springdemo.ioc.bean.Person@6be968ce	person
Person中自定义的初始化方法......
初始化 after -- 实例化的bean对象：com.fullstackboy.springdemo.ioc.bean.Person@6be968ce	person
com.fullstackboy.springdemo.ioc.bean.Person@6be968ce
```

