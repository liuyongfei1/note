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

