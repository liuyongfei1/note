## Spring静态代理

### 抛出使用场景

比如用户的增删改查4个方法，如果要对用户操作后进行日志记录，可能有人会说直接在用户增删改查后做日志记录就行。

一旦我想在用户操作之前加一个权限验证方法，那每个调用方法之前还得再加一个权限验证方法，这样的工作量是巨大的。

<img src="静态代理和动态.assets/image-20210731113326129.png" alt="image-20210731113326129" style="zoom:50%;" />

可能读者已经发现了这种做法的不足之处，如果有一天，蓝色背景的代码需要修改，

那是不是要同时修改三个地方?如果不仅仅是这三个地方包含这段代码，而是100个

，甚至是1000个地方，那会是什么后果?

### 弊端

- 混淆了业务方法本身的职责；
- 维护工作量巨大

### 解决方案

- 静态代理，将业务代理与记录日志、权限校验这些代码完全分开，实现松散耦合；
- 代理对象必须与被代理对象实现同一接口，在代理对象中实现记录日志的逻辑，并在需要的时候呼叫被代理对象，而被代理对象只保留业务代码。

#### 举例

##### 定义接口

```java
public interface IPerson {

         public abstract void sleep();

         public abstract void eating();

}
```

##### 被代理类

```java
public class Person implements IPerson {
  @Override
  public void sleep() {
    System.out.println("睡觉中");
  }
  @Override
  public void eating() {
    System.out.println("吃饭中");
  }
}
```

##### 代理类

```java
import org.apache.log4g.Logger;
public class IPersonProxy implement IPerson {
  private IPerson person;
  private Logger logger = Logger.getLogger(PersonProxy.class);
  
  public PersonProxy(IPerson person) {
    this.person = person;
  }
  @Override
  public void sleep() {
    // todo 增强逻辑
    logger.info("开始执行时间：" + new Date());
    person.sleep();
    logger.info("结束执行时间：" + new Date());
  }
  @Override
  public void eating() {
    // todo 增强逻辑
    logger.info("开始执行时间：" + new Date());
    person.eating();
    logger.info("结束执行时间：" + new Date());
  }
}
```

##### 测试类

```java
public class PersonTest {
		public static void main(String[]) args {
			IPersonProxy personProxy = new IPersonProxy(new Person);
			personProxy.sleep();
			personProxy.eating();
		}
}
```

### 弊端

一个代理接口只能服务一种类型的对象，对于稍大点的项目根本无法胜任。

#### 解决方案

使用动态代理。

## 动态代理

在程序运行过程中生成代理对象，由该代理对象去完成自己要去做的事情。



### 解决方案

将对象增删改查方法交给代理去执行，代理在执行方法前后可以做权限校验和日志记录。

### 代码demo

#### 定义接口

```java
/**
 * 定义接口
 * @date 2021/07/31
 */
public interface IPerson {

         public abstract void sleep();

         public abstract void eating();

}
```

接口实现类（被代理类）

```java
/**
 * 接口实现类
 * 只关注处理业务逻辑
 * @author Liuyongfei
 * @date 2021/7/31 12:08
 */
public class IPersonImpl implements IPerson{
    @Override
    public void sleep() {
        System.out.println("睡觉中");
    }

    @Override
    public void eating() {
        System.out.println("吃饭中");
    }
}
```

#### 处理器

```java
/**
 * 创建处理器，用于增强逻辑
 *
 * @author Liuyongfei
 * @date 2021/7/31 12:36
 */
public class DynaProxyHandler implements InvocationHandler {

    private static final Logger logger = LoggerFactory.getLogger(DynaProxyHandler.class);

    /**
     * 被代理对象
     */
    private Object target;
    public void setTarget(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("sleep")) {
            logger.info("sleep方法开始执行时间:" + new Date());
        } else {
            logger.info("eating方法开始执行时间:" + new Date());
        }

        Object result = method.invoke(target, args);

        if (method.getName().equals("sleep")) {
            logger.info("sleep方法结束执行时间:" + new Date());
        } else {
            logger.info("eating方法结束执行时间:" + new Date());
        }
        return result;
    }
}
```

动态代理工厂

```java
/**
 * 动态代理类工厂
 *
 * @author Liuyongfei
 * @date 2021/7/31 12:35
 */
public class DynamicProxyFactory {
    public static Object getProxy(Object object) {
        DynaProxyHandler handler = new DynaProxyHandler();
        handler.setTarget(object);

        return Proxy.newProxyInstance(object.getClass().getClassLoader(),
                object.getClass().getInterfaces(),handler);
    }
}
```

#### 测试

```java
/**
 * 动态代理
 *
 * @author Liuyongfei
 * @date 2021/7/31 10:40
 */
public class DynamicProxyTest {

    public static void main(String[] args) {
        IPerson person = (IPerson) DynamicProxyFactory.getProxy(new IPersonImpl());
        person.sleep();
        person.eating();
    }
}
```

### 总结

代理：本来应该自己做的事，由别人去做。

动态代理：在程序运行过程中生成代理对象，由代理对象去完成自己要去做的事情。

spring 的动态代理就是AOP的实现原理。

### SpringBoot动态代理有哪些

SpringBoot动态代理分为两种，JDK的动态代理和cglib的动态代理。

SpringBoot默认使用JDK的动态代理，当类没有实现接口时才使用cglib的动态代理。

#### jdk的动态代理

这种代理适用于实现了接口的类。

JDK动态代理的思想是：

生成一个类，让它和被代理的对象（UserServiceImpl）实现同样的接口，并重写接口的方法。

```java
public class DynamicProxyPerson {
  
}
```





#### cglib动态代理

jdk的动态代理只能代理实现了接口的类，但是有些类没有实现接口，那这种类如何被代理呢？

```java
//被代理类
public class Test {

    public void cal() {
        System.out.println("被代理方法");
    }
}
```

代理方式，自定义拦截器实现MethodInterceptor接口：

```java
public class AIntercepter implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        //增强逻辑
        System.out.println("代理前，增强逻辑");
        methodProxy.invokeSuper(o, objects);
        //增强逻辑
        System.out.println("代理后，增强逻辑");
        return null;
    }
}
```

测试：

```java
public class Main1 {

    public static void main(String[] args) {
        //代理类class文件存入本地磁盘，方便查看中间生成的临时文件
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\code");
        //------------------开始代理--------------------
        Enhancer enhancer = new Enhancer();
        //指定被代理类，生成的类将会继承此类
        enhancer.setSuperclass(Test.class);
        //指定拦截器
        enhancer.setCallback(new AIntercepter());
        //生成代理类
        Test t = (Test)enhancer.create();
        //完成代理
        t.cal();
        //------------------完成代理--------------------
    }
}
```

enhancer.create()方法会生成代理类，而System.setProperty()会将其保存到D:\code目录下。查看目录，会发现D:\code下生成了三个文件：

```java
Test$$EnhancerByCGLIB$$7ce11e59$$FastClassByCGLIB$$fe330774.class
Test$$EnhancerByCGLIB$$7ce11e59.class
Test$$FastClassByCGLIB$$3b8fb982.class
```

其中第二个文件是代理类，将其反编译，其中重要的方法展示如下：

```kotlin
//拦截其中的methodProxy.invokeSuper(o, objects);会调用此方法
final void CGLIB$cal$0() {
     super.cal();
}
//测试代码中的t.cal()会调用此方法
public final void cal() {
//获取自定义拦截器
 MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
 if (this.CGLIB$CALLBACK_0 == null) {
     CGLIB$BIND_CALLBACKS(this);
     var10000 = this.CGLIB$CALLBACK_0;
 }

 if (var10000 != null) {
       //调用自定义拦截器的intercept方法
     var10000.intercept(this, CGLIB$cal$0$Method, CGLIB$emptyArgs, CGLIB$cal$0$Proxy);
 } else {
     super.cal();
 }
}
```

整个过程的方法调用逻辑是：

Main1中的t.cal()   ------>  TestEnhancerByCGLIB7ce11e59.class中的 cal()   ------> 自定义拦截器AIntercepter中的intercept()   ------> 被代理类Test中的cal()方法。

