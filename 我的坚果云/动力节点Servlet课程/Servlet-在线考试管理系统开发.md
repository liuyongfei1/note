### 环境搭建

<img src="Servlet-在线考试管理系统开发.assets/image-20211109083718452.png" alt="image-20211109083718452" style="zoom:50%;" />

**在web工程中，jar包必须放在WEB-INF/lib目录（需要自己建lib目录）下面。这是一个固定位置，新加的jar包都需要放在这个目录下面。**

<img src="Servlet-在线考试管理系统开发.assets/image-20211109083900315.png" alt="image-20211109083900315" style="zoom:50%;" />

**设置 jar包，Add as Library**



### 用户信息注册流程图

![image-20211109085157984](Servlet-在线考试管理系统开发.assets/image-20211109085157984.png)



Servlet：

1. 调用请求对象，从请求头中，获取请求参数，得到用户的信息；
2. 调用DAO类，将用户信息填充到insert语句，借助JDBC协议，将数据插入到mysql服务器；
3. 调用响应对象，将处理结果以二进制形式写入到响应体中；
4. Tomcat服务器销毁请求对象和响应对象；
5. Tomcat服务器将Http响应协议包推送给发送请求的浏览器；
6. 浏览器根据响应头content-type指定编译器，对响应体二进制内容进行编辑；
7. 浏览器将处理结果展示给用户。 =》 过程结束。

其中第4步-第7步 是Tomcat服务器和浏览器自动完成的，不用我们手动干预处理。



### 用户添加开发

user_add.html 是网站中的一个静态资源文件，所以只能写在web目录下面，不能写在WEB-INF下面。

https://www.bilibili.com/video/BV1y5411p7kb?p=12&spm_id_from=pageDriver