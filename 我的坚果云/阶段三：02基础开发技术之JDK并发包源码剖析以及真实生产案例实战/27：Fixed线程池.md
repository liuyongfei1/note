### 关闭线程池

shutdown()方法会：

- 中断所有空闲的线程；
- 正在执行任务的线程会在执行完任务后再被关闭；
- 执行完队列中的任务；
- 所有的线程都会退出；
- 最后会关闭线程池；
- 就不能再提交新的任务了。

尝试将 Worker的state设置为1：

- 如果设置成功，则表明这个Worker当前的state =0，是处于空间的状态，没有执行任何一个任务，此时就可以中断这个Worker，Worker内部的线程就会退出。

- 如果设置失败，则表明这个Woker当前的state=1，正在执行任务，就不要去中断他了。



