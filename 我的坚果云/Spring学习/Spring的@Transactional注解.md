## 为什么加了@Transactional注解就支持事务了呢

Spring 注解类（包含Transactional）都是基于Spring AOP机制实现的。

### 总结

1. 如果Spring在Bean上检测到`@Transactional` 注解，它将创建Bean的动态代理类；
2. 代理类会访问事务管理器，事务管理器会打开/关闭事务连接；
3. 事务管理器本身其实就是执行你之前手动做的事：调用JDBC的api，去管理一个JDBC连接。





### 调用链

PurchaseOrderController.approve() 

->

@Service
@Transactional(rollbackFor = Exception.class)
public class PurchaseOrderServiceImpl implements PurchaseOrderService 

