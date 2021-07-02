### SpringBoot动态代理有哪些

SpringBoot动态代理分为两种，JDK的动态代理和cglib的动态代理。

SpringBoot默认使用JDK的动态代理，当类没有实现接口时才使用cglib的动态代理。

### jdk的动态代理

这种代理适用于实现了接口的类。

假设现在有一个接口UserService：

```java
public interface UserService {
	void query(String name);
}
```

有一个类UserServiceImpl，实现了接口：

```java
// 被代理的类
public class UserService implements UserService {
	@Override
	public void query(String name) {
		System.out.println("query name = " + name);
	}
}
```

那么怎么代理上面这个UserServiceImpl类呢？

JDK动态代理的思想是：

生成一个类，让它和被代理的对象（UserServiceImpl）实现同样的接口，并重写接口的方法。

如下所示（注意：下面这个类是JDK自动生成的）：

