同一个网站（myWeb）下，给同一用户提供服务，OneServlet将数据传给TwoServlet。

OneServlet：

```java
// 从请求头获取参数信息
doGet(HttpServletRequest request, HttpServletResponse response) {
	 HttpSession session = request.getSession();
  
  session.setAttribute("key1","共享数据");
}
```

TwoServlet：

```java
// 从请求头获取参数信息
doGet(HttpServletRequest request, HttpServletResponse response) {
	 HttpSession session = request.getSession();
  
   Object data = session.getAttribute("key1");
}
```

##### HttpSession与用户关联的原理

- 用户第一次访问浏览器；

- Tomcat在创建一个HttpSession时，会自动为这个HttpSession对象生成一个唯一编号；

- Tomcat将编号保存到Cookie对象，推送到当前浏览器缓存。Cookie：JSESSIONID= *********

- 用户第二次访问浏览器时，会在请求头把这个Cookie带过去；

- Tomcat根据请求头JSESSIONID确认用户是否有HttpSession对象以及哪一个HttpSession对象是当前用户。


跟去洗浴中心洗澡时，储物柜的那个手牌 是一样的作用。

##### HttpSession销毁时机

- 用户与HttpSession关联时使用的这个Cookie，只会存在浏览器缓存，不会存在浏览器所在客户端磁盘上；
- 在浏览器被关闭时，意味着用户与他的HttpSession关系被切断；
- 由于Tomcat无法检测浏览器何时关闭，因此在此时关闭浏览器并不会导致Tomcat将浏览器关联的HttpSession销毁；
- 为了解决这个问题，Tomcat会为每个HttpSession对象设置一个【空闲时间】，这个空闲时间默认是30分钟。如果当前的HttpSession对象空闲时间超过30分钟，就会被Tomcat销毁掉。

##### HttpSession空闲时间手动设置

在当前网站web/WEB-INF/web.xml：

```xml
<session-config>
  <!--设置当前网站中每一个HttpSession的空闲时间为5分钟-->
	<session-timeout>5</session-timeout>
</session-config>
```



https://www.bilibili.com/video/BV1y5411p7kb?p=28





