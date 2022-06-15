### 1. synchronized的编译后产物

- 修饰方法时候，编译器会生成ACC_SYNCHRONIZED关键字来标识
- 修饰代码块的时候，会依赖monitorenter和monitorexit指令

但是无论是方法还是代码块，对应的锁都是一个实例

### 2. synchronized和对象头关系

在内存中，对象一般由三个部分组成，对象头、对象实际数据和对齐填充。对象头分为几个部分组成，但是sync仅和对象头里的Mark Word信息有关。

Mark Word信息会记录对象关于锁的信息。又因为每个对象都会有一个与之对应的monitor对象，monitor对象中存储者当前持有锁的线程以及等待
锁的线程队列。

![Sync、Mark Word以及Monitor的关系](../Images/sync_mark_word_monitor.png)

### 3. synchronized在1.6版本之后的优化

#### 1.6之前

- 是重量级锁
- 线程进入同步代码块、方法时，monitor对象就会把当前线程id进行记录，设置Mark Word的monitor地址，并把阻塞的
线程存储到monitor的等待队列中
- 加锁依赖操作系统的mutex指令实现，所以会有用户态、内核态的切换，十分浪费性能

#### 1.6之后

引入了偏向锁和轻量级锁在JVM层，避免了操作系统级别的切换，没有切换的消耗。Mark Down对锁的记录一种有四种：无锁、偏向锁、轻量级锁、重量级锁

- 偏向锁：JVM认为只有某个线程才会执行同步代码。Mark Word记录线程的id，如果线程id一致就能获取到锁，执行代码。如果不一致，则通过CAS修改线程
id，如果修改成功，就能获取到锁并执行同步代码；如果失败则升级到轻量级锁

- 轻量级锁: 当前线程会在栈帧下创建Lock Record，Lock Record会把Mark Word信息拷贝进去，且有个Owner指针指向加锁的对象。线程执行到同步代码块时，
则用CAS视图将Mark Word的指针指向到当前线程栈帧的Lock Record地址，如果修改成功，则获取到轻量级锁；如果失败则升级到重量级锁

- 重量级锁：操作系统mutex实现

![Sync的锁分类.png](../Images/sync_lock_sort.png)

#### 偏向锁、轻量级锁、重量级锁的场景

- 当一个线程进入临界区，偏向锁
- 当多个线程交替进入临界区，轻量级锁
- 当多个线程同时进入临界区，重量级锁


