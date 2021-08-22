### Atomic原理图

<img src="原子类技术体系：AutomicInteger.assets/Atomic原理.png" alt="Atomic原理" style="zoom:50%;" />

Atomic原子类体系的底层核心的原理是CAS。Compare And Set。

无锁化。

- 每次尝试修改值的时候，就对比一下，有没有人修改过这个值，没有人修改的话，就自己修改；
- 如果有人修改过，就重新查出来最新的值；
- 再次重复这个过程。

