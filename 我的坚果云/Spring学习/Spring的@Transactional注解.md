## Spring为什么加了@Transactional注解就支持事务了呢

Spring 注解类（包含`@Transactional`）都是基于Spring AOP机制实现的。

### 概述

1. 如果Spring在Bean上检测到`@Transactional` 注解，它将创建Bean的动态代理类；
2. 代理类会访问事务管理器，事务管理器会打开/关闭事务连接；
3. 事务管理器本身其实就是执行你之前手动做的事：调用JDBC的api，去管理一个JDBC连接。

### Spring源码@Transactional的调用链

![Spring的@Transtractional注解](Spring的@Transactional注解.assets/Spring的@Transtractional注解.png)

备注：

1. 该调用链对应源码Spring 4.3.13版本；
2. 关键点已在图中加粗标注。

### 总结

- 当我们的应用系统在调用那些声明了`@Transactional`的目标类/方法时，Spring会使用AOP代理，在代码运行时生成一个代理对象；
- @Transactional的目标类/方法会被`TransactionInterceptor` 事务拦截器进行拦截；
- 在目标方法开始执行之前加入事务；
- 执行目标方法的业务逻辑；
- 根据执行情况是否出现异常，利用事务管理器`AbstractPlatformTransactionManager`来操作数据源DataSource，进行事务提交或事务回滚。

#### 涉及jar包

通过上图可以清晰的看到一个详细的源码调用链，一共涉及到了以下几个核心jar包：

1. spring-aop；
2. spring-tx；
3. spring-orm；
4. hibernate-entitymanager；
5. hibernate-core。

#### 涉及概念

涉及到的核心概念如下：

1. 动态代理；
2. AOP；
3. 编程式事务和声明式事务；
4. 事务管理器；
5. 事务传播级别；
6. 事务隔离级别；
7. hibernate orm；
8. JDBC。



### Spring配置事务的几种方式

spring配置文件中关于事务配置总是由3个组成部分：DataSource，TransactionManager，代理机制这三个部分。

拦截器

1. 定义数据源；
2. 定义会话工厂，使用定义的数据源；
3. 定义事务管理器，使用定义的会话工厂；
4. 定义服务层的bean，使用会话工厂。

类似这样：

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xmlns:aop="http://www.springframework.org/schema/aop"  
    xmlns:tx="http://www.springframework.org/schema/tx"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd  
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd  
           http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd">  
    <!-- 数据源 -->  
    <bean id="dataSource"  
        class="org.apache.commons.dbcp.BasicDataSource"  
        destroy-method="close">  
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />  
        <property name="url"  
            value="jdbc:mysql://192.168.0.244:3306/test?useUnicode=true&amp;characterEncoding=UTF-8" />  
        <property name="username" value="root" />  
        <property name="password" value="root" />  
        <!-- 连接池启动时的初始值 -->  
        <property name="initialSize" value="10" />  
        <!-- 连接池的最大值 -->  
        <property name="maxActive" value="10" />  
        <!-- 最大空闲值.当经过一个高峰时间后，连接池可以慢慢将已经用不到的连接慢慢释放一部分，一直减少到maxIdle为止 -->  
        <property name="maxIdle" value="20" />  
        <!--  最小空闲值.当空闲的连接数少于阀值时，连接池就会预申请去一些连接，以免洪峰来时来不及申请 -->  
        <property name="minIdle" value="10" />  
        <property name="defaultAutoCommit" value="true" />  
    </bean>  
    <!-- 会话工厂 -->  
    <bean id="sessionFactory"  
        class="org.springframework.orm.hibernate3.LocalSessionFactoryBean">  
        <property name="dataSource" ref="dataSource" />  
        <property name="mappingLocations">  
            <list>  
                <value>classpath:/com/nms/entity/**/*.hbm.xml</value>  
            </list>  
        </property>  
        <property name="hibernateProperties">  
            <props>  
                <prop key="hibernate.dialect">  
                    org.hibernate.dialect.MySQL5Dialect  
                </prop>  
                <prop key="hibernate.show_sql">true</prop>  
                <prop key="hibernate.format_sql">true</prop>  
            </props>  
        </property>  
    </bean>      
     <!-- 定义事务管理器（声明式的事务） -->    
    <bean id="transactionManager"  
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">  
        <property name="sessionFactory" ref="sessionFactory" />  
    </bean>     
     <!-- 定义事务 -->   
    <bean id="transactionInterceptor"    
        class="org.springframework.transaction.interceptor.TransactionInterceptor">    
        <property name="transactionManager" ref="transactionManager" />    
        <!-- 配置事务属性 -->    
        <property name="transactionAttributes">    
            <props>    
                <prop key="*">PROPAGATION_REQUIRED</prop>    
            </props>    
        </property>    
    </bean>        
    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">    
        <property name="beanNames">    
            <list>    
                <value>*DaoImpl</value>  
            </list>    
        </property>    
        <property name="interceptorNames">    
            <list>    
                <value>transactionInterceptor</value>    
            </list>    
        </property>    
    </bean>    
    <!-- 配置服务层 -->  
    <bean id="userDaoAgency" class="com.dao.impl.UserDaoImpl">  
        <property name="sessionFactory" ref="sessionFactory" />  
    </bean>  
</beans>  
```

