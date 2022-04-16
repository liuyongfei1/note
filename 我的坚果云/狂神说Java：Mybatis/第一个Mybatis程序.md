### 思路

搭建环境 -》 导入Mybatis -》编写代码 -》测试。

### 1、搭建环境

#### 1.1搭建数据库

启动数据库，创建库，创建表，插入数据。

#### 1.2 创建项目

建一个maven工程，blog-demo/mybatis-demo。

导入maven依赖。

### 2.1 创建MybatisUtil工具类

```java
/**
 * Mybatis工具类
 *
 * @author Liuyongfei
 * @date 2021/11/27 16:34
 */
public class MybatisUtil {

    private static SqlSessionFactory sqlSessionFactory;

    static {

        try {
            // 使用Mybatis第一步，获取SqlSessionFactory对象
            String resource = "mybatis-config.xml";
            InputStream inputStream = Resources.getResourceAsStream(resource);
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 有了SqlSessionFactory，就可以获取SqlSession实例
     * 获取SqlSession实例,SqlSession实例包含了面向数据库执行sql命令的各种方法
     * @return SqlSession实例
     */
    public static SqlSession getSqlSession() {
        return sqlSessionFactory.openSession();
    }
}
```

### 2.3 编写代码

- 实体类
- DAO接口
- 接口实现类

#### 2.4 测试

具体代码见 blog-demo/mybatis-demo。

```java
/**
 * 用户DAO类的单元测试类
 *
 * @author Liuyongfei
 * @date 2021/11/27 17:49
 */
public class TestUserDAO {

    @Test
    public void getUserListTest() {
        SqlSession sqlSession = MybatisUtil.getSqlSession();

        UserDAO mapper = sqlSession.getMapper(UserDAO.class);
        List<User> list = mapper.getUserList();

        for (User user : list) {
            System.out.println(user);
        }

    }
}
```

输出：

```bash
User(id=1, name=张三, pwd=123)
User(id=2, name=李四, pwd=567)
User(id=3, name=王五, pwd=789)
```

### 为什么Mybatis不用实现类，只写接口就可以执行SQL

看一下源码。

1、SqlSessionFactoryBuilder#build：

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        SqlSessionFactory var5;
        try {
            XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
            var5 = this.build(parser.parse());
        } catch (Exception var14) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", var14);
        } finally {
            ErrorContext.instance().reset();

            try {
                inputStream.close();
            } catch (IOException var13) {
            }

        }

        return var5;
    }
```

2、XMLConfigBuilder：

```java
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
        super(new Configuration());
        ErrorContext.instance().resource("SQL Mapper Configuration");
        this.configuration.setVariables(props);
        this.parsed = false;
        this.environment = environment;
        this.parser = parser;
    }
```



3、Configuration类的addMapper：

```java
public <T> void addMapper(Class<T> type) {
        this.mapperRegistry.addMapper(type);
    }
```

4、MapperRegistry#addMapper：

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (this.hasMapper(type)) {
            throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }

        boolean loadCompleted = false;

        try {
            this.knownMappers.put(type, new MapperProxyFactory(type));
            MapperAnnotationBuilder parser = new MapperAnnotationBuilder(this.config, type);
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                this.knownMappers.remove(type);
            }

        }
    }

}

public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory)this.knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        } else {
            try {
                return mapperProxyFactory.newInstance(sqlSession);
            } catch (Exception var5) {
                throw new BindingException("Error getting mapper instance. Cause: " + var5, var5);
            }
        }
    }
```

看到这个 MapperProxyFactory 类的名字，就知道这个 Mapper 代理类工厂。

5、MapperProxyFactory

```java
public T newInstance(SqlSession sqlSession) {
    MapperProxy<T> mapperProxy = new MapperProxy(sqlSession, this.mapperInterface, this.methodCache);
    return this.newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
        return Proxy.newProxyInstance(this.mapperInterface.getClassLoader(), new Class[]{this.mapperInterface}, mapperProxy);
    }
```

看到了熟悉的Proxy.newProxyInstance（jdk动态代理），还差一个 invocationHandler，可以看到对应的位置是MapperProxy。

6、MapperProxy

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
        try {
            return method.invoke(this, args);
        } catch (Throwable var5) {
            throw ExceptionUtil.unwrapThrowable(var5);
        }
    } else {
        MapperMethod mapperMethod = this.cachedMapperMethod(method);
        return mapperMethod.execute(this.sqlSession, args);
    }
}
```

这样动态代理实现类就完成了，mapper接口方法调用时，会触发代理类的 invoke 方法。

剩余的工作就是根据 类.方法名 执行sql语句了。