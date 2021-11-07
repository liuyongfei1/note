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

### 四、HttpServletResponse接口

#### 介绍

1. HttpServletResponse接口来自于servlet规范中，在Tomcat的servlet-api.jar中；
2. HttpServletResponse接口实现类由Http服务器提供；
3. HttpServletResponse接口实现类负责将doGet或doPost方法执行结果写入到【响应体】交给浏览器；
4. 开发人员习惯将HttpServletResponse接口修改的对象称为响应对象。

#### 主要功能

1. 将执行结果以二进制形式写到【响应体】中；
2. 设置响应头中【content-type】属性值，从而控制浏览器使用对应编译器将接收到的二进制数据编译为：【文字、图片、视频、命令】；
3. 设置响应头中【location】属性，将一个请求地址赋值给location，从而控制浏览器向指定服务器发送请求。

以Hello World测试：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>HelloWorldServlet</servlet-name>
        <servlet-class>com.fullstackboy.servlet.controller.HelloWorldServlet</servlet-class>
        <!--在tomcat接收到第一次对该servlet实例对象请求时才会创建实例对象-->
    </servlet>

    <!--为servlet设置别名-->
    <servlet-mapping>
        <servlet-name>HelloWorldServlet</servlet-name>
        <url-pattern>/three</url-pattern>
    </servlet-mapping>
</web-app>
```

HelloWorldServlet：

```java
public class HelloWorldServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request,  HttpServletResponse response) throws javax.servlet.ServletException, IOException {
        // 模拟业务逻辑处理结果
        String result = "Hello, world!";

        // 1.通过响应对象，向Tomcat索要输出流
        PrintWriter out = response.getWriter();

        // 2.通过输出流将数据以二进制形式发送给【响应体】
        // out.write(result);
        out.print(result);
        // doGet执行完毕，Tomcat将响应体推送给浏览器
    }
}
```

<img src="Servlet.assets/image-20211107185351393.png" alt="image-20211107185351393" style="zoom:50%;" />



浏览器的请求行为即可以由前端工程师来控制，也可以由后端工程师来控制。

可以通过服务端来控制浏览器的请求行为。怎么做呢？在这里就体现到了。

#### 修改content-type属性

```java
public class FourServlet extends HttpServlet {

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
//        String result = "Java<br/>Mysql<br/>Vue";
        String result = "你好";

        // 默认content-type=text，这样的浏览器不会解析html字符
//        response.setContentType("text/html");
        response.setContentType("text/html;charset=utf-8");

        // 模拟业务逻辑处理结果

        // 1.通过响应对象，向Tomcat索要输出流
        PrintWriter out = response.getWriter();

        // 2.通过输出流将数据以二进制形式发送给【响应体】
        out.print(result);

        // doGet执行完毕，Tomcat将响应体推送给浏览器
    }
}
```



#### sendRedirect方法

// 通过响应对象，将地址赋值给响应头中的location属性，sendRedirect方法远程控制浏览器请求行为【请求地址、请求方式、请求参数】。

```java
public class FiveServlet extends HttpServlet {


    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String result = "http://www.baidu.com";

        // 通过响应对象，将地址赋值给响应头中的location属性
        response.sendRedirect(result);

        // 浏览器在收到响应后，如果发现响应头中有location属性，会自动通过地址栏向location指定的网站发送请求
        // sendRedirect方法远程控制浏览器请求行为【请求地址、请求方式、请求参数】
    }
}
```

