## redisson的同一客户端同一线程读写锁

### 同一个客户端同一线程，先加读锁再加读锁

先加了一次读锁：

```
anyLock: {
	"mode" : "read",
	"UUID_01_thread_01": 1
}
{anyLock}:UUID_01:thread_01:rwlock_timeout:1 1
```

再来一次读锁：

hget anyLock mode = read

由于之前已经加过读锁，因此这里返回true，执行：

```
hincrby anyLock UUID_01:thread_01 1
set {anyLock}:UUID_01:thread_01:rw_timeout:2 1
pexpire anyLock 30000
pexpire {anyLock}:UUID_01:thread_01:rw_timeout:2 30000 
```

生成的数据结构为：

```
anyLock: {
	"mode": "read",
	"UUID_02:thread_02": 2
}
{anyLock}:UUID_01:thread_01:rwlock:timeout:2 1
```

#### 结论

同一个客户端同一线程可以多次加读锁。

### 同一个客户端同一线程，先加读锁再加写锁

先加了一次读锁：

```
anyLock: {
	"mode": "read",
	"UUID_01:thread_01": 1
}
{anyLock}:UUID_01:thread_01:rwlock:timeout:1 1
```

#### 参数

再来尝试加写锁：

- KEYS[1]：anyLock
- ARGV[1]：30000
- ARGV[2]：{anyLock}:UUID_01:thread_01:write

#### lua脚本

执行

```
hget anyLock mode
```

由于之前加的是读锁，所以这里返回 read，第一个判断条件和第二个判断条件都不成立，直接返回：

```
pttl anyLock
```

直接返回anyLock的剩余生存时间，导致加锁失败。

#### 结论

同一个客户端同一个线程先加读锁再加写锁是互斥的。

