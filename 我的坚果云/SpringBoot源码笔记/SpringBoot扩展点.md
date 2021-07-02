## 列举一两个SpringBoot启动的时候的扩展点

### 监听容器刷新完成扩展点：ApplicationListener<ContextRefreshedEvent>

1. 熟悉Spring的同学知道，容器刷新成功意味着所有的Bean初始化已经完成。

2. 当容器刷新之后Spring将会调用容器内所有实现了ApplicationListener<ContextRefreshedEvent>的Bean的onApplicationEvent方法。

3. 应用程序可以以此达到监听容器初始化完成事件的目的。

```java
@Component
public class StartupApplicationListenerExample implements ApplicationListener<ContextRefreshedEvent> {
  private static final Logger LOG
      = Logger.getLogger(StartupApplicationListenerExample.class);
  public static int counter;
  
  @Override
  public void onApplicationEvent(ContextRereshedEvent event) {
     LOG.info("Increment counter");
     counter++;
  }
}
```

易错的点：

这个扩展点用在`web`容器中的时候需要额外注意，在web [项目]()中（例如`spring mvc`），系统会存在两个容器，一个是`root application context`,另一个就是我们自己的`context`（作为`root application context`的子容器）。如果按照上面这种写法，就会造成`onApplicationEvent`方法被执行两次。

解决此问题的方法如下：

```java
@Component
public class StartupApplicationListenerExample implements ApplicationListener<ContextRefreshedEvent> {
  private static final Logger LOG
      = Logger.getLogger(StartupApplicationListenerExample.class);
  public static int counter;
  
  @Override
  public void onApplicationEvent(ContextRereshedEvent event) {
  	if (event.getApplicationContext().getParent() == null) {
  		LOG.info("Increment counter");
     counter++;
  	}
  }
}
```



### SpringBoot的CommandLineRunner接口

当容器上下文初始化完成后，SpringBoot也会调用所有实现了CommandLineRunner接口的run方法，下面这段代码可以起到和上文同样的作用：

```java
@Component
public class CommandLineAppStartupRunner implements CommandLineRunner {
  private static final Logger LOG
      = Logger.getLogger(CommandLineAppStartupRunner.class);
  public static int counter;
  
  @Override
  public void run(String...args) {
  	LOG.info("Increment counter");
    counter++;
  }
}
```

