### 原因

对本地数据库的操作是纯事务，不应该将本地数据库操作跟远程接口的调用，混在一块儿。

可能会出现莫名其妙的事务之间的锁争用的问题。**通过分布式系统触发了一个分布式系统场景下的问题。**

### 调用链路

采购服务：审核采购单，更新采购单状态后，通知调度中心  =》调度服务：调度采购入库 =》 wms服务：创建采购入库单。

### 错误浮现

使用postman请求 库存服务的 审核采购单接口：

<img src="添加大量日志来定位wms服务调用采购服务超时的问题.assets/image-20210718162428739.png" alt="image-20210718162428739" style="zoom:50%;" />

库存服务报错如下：

```java
Caused by: com.netflix.client.ClientException: Load balancer does not have available server for client: eshop-schedule
at com.netflix.loadbalancer.LoadBalancerContext.getServerFromLoadBalancer(LoadBalancerContext.java:483) ~[ribbon-loadbalancer-2.2.5.jar:2.2.5]
```

