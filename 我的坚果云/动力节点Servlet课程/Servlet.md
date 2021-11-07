## Servlet

### 一、Servlet规范

- servlet规范来自于 javaee规范的一种；
- 作用：
  - 在servlet规范中，指定【动态资源文件】的开发步骤；
  - 在servlet规范中，指定Http服务器调用【动态资源文件】的规则；
  - 在servlet规范中，指定Http服务器管理动态资源文件实例对象规则。

### 二、Servlet接口实现类

- servlet接口来自于servlet规范下的一个接口，这个接口存在于Http服务器，提供jar包；
- Tomcat服务器下lib目录有一个servlet-api.jar存放Servlet接口：

### 三、Servlet对象生命周期(http服务器是怎么管理这个servlet对象)

1. 网站中所有servlet接口实现类的实例对象，只能由Http服务器负责创建。开发人员不能手动创建servlet接口实现类的实例对象。
2. 在默认情况下，Http服务器接收到对于当前servlet接口实现类第一次请求时，会自动创建这个servlet接口实现类实例对象。
3. 在http服务器运行期间，一个servlet接口实现类只能被创建一个实例对象。一个进程对应多个线程，用户请求相当于一个线程，无论有多少用户访问tomcat这个servlet接口实现类，自始至终，tomcat只会new一次这个实例对象。
4. 这个servlet接口实现类对象什么时候被销毁呢？=》在http服务器关闭的时候，会自动将网站中所有的sevelet对象销毁。

用java web的servlet接口实现类demo（blog-demo/servlet-life-cycle），来了解servlet对象声明周期：

- TwoServlet 会在tomcat初始化的过程中就被创建；

- OneServlet是在请求http://localhost:13000/myWeb/one时，会被创建：

  ```bash
  OneServlet 实例对象被创建......
  OneServlet doGet is run...
  ```

  再次访问：

  <img src="Servlet.assets/image-20211107173039480.png" alt="image-20211107173039480" style="zoom:50%;" />

​       会发现，OneServlet不会再被创建了。也就是不管被访问多少次，只会被创建一个实例对象。

