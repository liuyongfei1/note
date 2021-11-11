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

HttpSession与用户关联的原理：

- 用户第一次访问浏览器；

- Tomcat在创建一个HttpSession时，会自动为这个HttpSession对象生成一个唯一编号（这个编号只会存在浏览器缓存，不会存在浏览器所在客户端磁盘上）；

- Tomcat将编号保存到Cookie对象，推送到当前浏览器缓存。Cookie：JSESSIONID= *********

- 用户第二次访问浏览器时，会在请求头把这个Cookie带过去；

- Tomcat根据请求头JSESSIONID确认用户是否有HttpSession对象以及哪一个HttpSession对象是当前用户。

  

跟去洗浴中心洗澡时，储物柜的那个手牌 是一样的作用。

https://www.bilibili.com/video/BV1y5411p7kb?p=28





