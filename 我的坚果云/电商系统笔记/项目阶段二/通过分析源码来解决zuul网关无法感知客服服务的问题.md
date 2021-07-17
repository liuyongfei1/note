### 场景

明明客服服务启动了，eureka注册中心里也是有的，但是zuul网关却始终感知不到客服服。

如果在公司里投入生产环境使用的时候，碰到了这个问题怎么办呢?

源码级别定位来解决这个问题。

ribbon从eureka里获取service list的代码

看spring-cloud-netflix-eureka-client 工程里的

DomainExtractingServerList.java => DiscoveryEnableNIWSServerList.java

DiscoveryClient.java里的 localRegionApps