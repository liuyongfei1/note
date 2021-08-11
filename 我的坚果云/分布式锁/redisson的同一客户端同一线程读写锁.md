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

### 同一个客户端同一线程，先加写锁再加读锁

先加一次写锁：

```
anyLock: {
	"mode": "write",
	"UUID_01:thread_01:write": 1
}
```

再来加一次读锁：

#### 参数

- KEYS[1]：anyLock
- KEYS[2]：{anyLock}:UUID_01:thread_01:rwlock_timeout
- ARGV[1]：30000毫秒
- ARGV[2]：UUID_01:thread_01
- ARGV[3]：UUID_01:thread_01:write

#### 脚本解释

执行

```
hget anyLock mode
```

由于之前加的是写锁，所以这里返回write，看第二个判断条件：

```lua
"if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then " +
                                  "local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                                  "local key = KEYS[2] .. ':' .. ind;" +
                                  "redis.call('set', key, 1); " +
                                  "redis.call('pexpire', key, ARGV[1]); " +
                                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                  "return nil; " +
                                "end;"
```

hexists anyLock UUID_01:thread_01 存在，也就是说是同一个客户端同一个线程，所以返回1，继续执行：

```
hincrby anyLock UUID_01:thread_01 1

set {anyLock}:UUID_01:thread_01:rwlock_timeout:1 1

pexpire anyLock 30000

pexpire  {anyLock}:UUID_01:thread_01:rwlock_timeout 30000
```

返回 nil，加锁成功。

#### 结论

**同一个客户端同一个线程，先加了一个写锁，再加读锁是可以成功的，也就是说默认在同一个线程写锁的期间，可以多次加读锁。**

### 同一个客户端同一个线程先加写锁，再加写锁

先加一次写锁：

```
anyLock: {
	 "mode": "write",
	"UUID_01:thread_01:write" : 1,
}
```

再来加一次写锁：

```lua
"if (mode == 'write') then " +
    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
        "local currentExpire = redis.call('pttl', KEYS[1]); " +
        "redis.call('pexpire', KEYS[1], currentExpire + ARGV[1]); " +
        "return nil; " +
    "end; " +
  "end;"
```

hexists anyLock UUID_01:thread_01:write ，存在返回1，继续执行：

```
hincrby anyLock UUID_01:thread_01:write 1
pexpire anyLock 50000
```

生成新的数据结构：

```
anyLock: {
	"mode": "write",
	"UUID_01:thread_01:write": 2
}
```

然后返回nil，加锁成功。

#### 结论

可以看到，同一客户端同一线程是可以重入加锁的，读写锁其实也是一种可重入锁。