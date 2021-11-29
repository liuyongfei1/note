#### IOC理论推导

- Hello对象是谁创建的？
  是由Spring创建的。
- Hello对象的属性是怎么设置的？
  是由Spring设置的。

这个过程叫控制反转。

控制：谁来控制对象的创建，传统的应用程序是由程序本身来控制的，使用Spring后，对象是由Spring来控制的。

反转：程序本身不创建对象，而变成被动的接收对象。

依赖注入：就是利用set方法来进行注入的。



IOC是一种编程思想，由主动的编程变成被动的接收。

到了现在，我们不用在程序中去改动了代码了，要实现不同的操作，只需在xml配置文件中进行修改。

所谓的IOC，一句话搞定：对象由Spring来创建，管理，装配。

https://www.bilibili.com/video/BV1WE411d7Dv/?p=3&spm_id_from=pageDriver

