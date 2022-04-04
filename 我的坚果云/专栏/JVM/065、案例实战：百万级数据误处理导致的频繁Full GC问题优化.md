### 1、线上故障场景

有次一个线上系统进行了一次版本升级，结果升级过后才半个小时，突然就收到运营和客服雪花般的反馈，说这个系统对应的前端页面无法访问了，所有用户看到的全是一片空白和错误信息。

这个时候通过监控报警平台也收到雪花般的报警，发现线上系统所在机器的CPU负责非常高，持续走高，甚至直接导致机器都宕机了。所以系统对应的前端页面当然是什么都看不到了。

### 2、demo案例

#### demo

见/Users/lyf/Workspace/www/blog-demo/jvm-demo的 CreateBigObject。

#### 生成内存快照

```bash
 ✘ 👍 @MacBook-Pro   ~/Workspace/www/blog-demo   develop  jps
97617 Resin
97616 Launcher
96199 RemoteMavenServer36
99335 Launcher
12423 Launcher
12424 CreateBigObject
13848 Jps
96056
99340 Resin
 👍 @MacBook-Pro   ~/Workspace/www/blog-demo   develop  sudo -u lyf jmap -dump:format=b,file=heap2.hprof 12424
Dumping heap to /Users/lyf/Workspace/www/blog-demo/heap2.hprof ...
Heap dump file created
```

#### 使用MAT

使用MAT打开生成的.hprof文件，可以进行分析。