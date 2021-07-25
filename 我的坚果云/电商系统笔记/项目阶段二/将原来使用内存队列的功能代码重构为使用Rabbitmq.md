### 重构库存服务和调度服务的消息队列代码

#### 重构说明

使用Rabbitmq消息队列 + spring cloud stream。

#### 库存服务

通知库存中心，"提交订单事件发生了"：

```java
informSubmitOrderEvent(OrderInfoDTO orderInfoDTO) {
  try {
      // ...... 
      // 发送异步消息到内存队列
      GoodsStockUpdateMessage message = new GoodsStockUpdateMessage();
      message.setId(UUID.randomUUID().toString().replace("-",""));
      message.setOperation(GoodsStockUpdateOperation.SUBMIT_ORDER);
      message.setParameter(orderInfoDTO);

      // 改为使用rabbitmq消息中间件
      String messageJson = JSON.toJSONString(message);
      Message msg = MessageBuilder.withPayload(messageJson.getBytes()).build();
      messageService.stockUpdateMessageQueue().send(msg);
        } catch (Exception e) {
            logger.error("error",e);
            return false;
        }
        return true;
    }
```

#### 调度服务

在Spring启动类里添加接收消息方法：

```java
@SpringBootApplication
@EnableScheduling
@ServletComponentScan
@EnableEurekaClient
@EnableFeignClients
@EnableBinding(MessageService.class)
@Import(DruidDataSourceConfig.class)
public class ScheduleApplication {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(ScheduleApplication.class, args);
        SpringContext.setContext(context);
    }

    @StreamListener("stock-update-message-queue")
    public void receive(byte[] message) {
        ScheduleStockUpdateMessageConsumer consumer =
                SpringContext.getContext().getBean(ScheduleStockUpdateMessageConsumer.class);
        consumer.consume(new String(message));
    }
}
```

消费者直接消费消息：

```java
@Component
public class ScheduleStockUpdateMessageConsumer {
  /**
     * 消费库存更新消息
     */
    public void consume(String messageJson) {
        try {
            // 从队列中取出消息
            GoodsStockUpdateMessage message = JSONObject.parseObject(messageJson, GoodsStockUpdateMessage.class);

            if (!isOrderRelatedMessage(message)) {
                return;
            }

            // 处理消息
            OrderInfoDTO order = getOrderFromMessage(message);
            processMessage(message, order);
        } catch (Exception e) {
            logger.error("error", e);
        }
    }
}
```