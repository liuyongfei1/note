## redisson的读锁非互斥

### 读锁非互斥

客户端A，客户端B，两个客户端部署在不同的机器上，都要对anyLock这个锁加读锁，此时是不会互斥的，都可以加锁成功。

### lua脚本分析

假设客户端A(UUID_01:thread_01)先加了一把读锁：

```
anyLock: {
	"mode": "read",
	"UUID_01:thread_01": 1
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
```

此时客户端B（UUID_02:thread_02）也要来加读锁，最后的存储结果是：

```
anyLock: {
	"mode": "read",
	"UUID_01:thread_01": 1,
	"UUID_02:thread_02": 1
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
{anyLock}:UUID_02:thread_02:rwlock_timeout:1 1
```

加锁多次，则为：

{anyLock}:UUID_01:thread_01:rwlock_timeout:5 1

等

同时：

```
pexpire anyLock 30000
pexpire {anyLock}:UUID_02:thread_02:rwlock_timeout:1 30000
```

### 总结

- 多个客户端同时加读锁，读锁与读锁是不会互斥的，只会不断的在hash结构里加入那个客户端的一条数据；
- 每个客户端都会维持一个watchdog，不断的刷新anyLock的存活时间，同时也会刷新那个客户端对应自己的time out的生存时间。

