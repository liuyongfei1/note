### 重构库存服务和调度服务的消息队列代码

#### 库存服务

通知库存中心，"提交订单事件发生了"：

```java
informSubmitOrderEvent(OrderInfoDTO orderInfoDTO)
```

