## 为什么加了@Transactional注解就支持事务了呢

Spring 注解类（包含Transactional）都是基于Spring AOP机制实现的。



### 调用链

PurchaseOrderController.approve() 

->

@Service
@Transactional(rollbackFor = Exception.class)
public class PurchaseOrderServiceImpl implements PurchaseOrderService 

