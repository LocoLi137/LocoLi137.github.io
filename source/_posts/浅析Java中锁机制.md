---
title: 浅析Java中锁机制
date: 2023-12-06 11:13:09
tags:
---
- 思想
    - 乐观锁
        - AQS
    - 悲观锁
        - ReentrantLock
        - Syschronized



# 一、乐观锁

### 1、是什么

CAS(compare and swap)

(内存位置，预期原值，预期新值)

保证内存可见的前提，在修改变量前对变量进行预期值对比，若相等则修改，若不等则自旋重试。

**注**：需要使用volatile修饰，保证可见性

### 2、怎么玩

自旋锁

Atomic包

LongAddr:原子累加，分段锁思想，每段的内部还是cas，算是atomic的优化

AQS:期望获取锁的线程会先用cas的方式修改共享变量state（不同的同步器下有不同含义）

甚至数据库中版本号字段管理

### 3、原理

CAS→UNSAFE→CompareAndSwap(X1,X2,X3,X4)【inter x86 cpu原子指令→cmpxchg指令，对总线加锁保证原子性，硬件层面实现】

TODO：后续此处链接一片JMM

### 4、缺陷

CPU空转→睡一会

ABA→版本号



# 二、悲观锁

特点：当线程获取不到锁会被阻塞

- ReentrantLock→api层面
- synchronized→jvm层面

### 1、synchronized

#### **延伸**：对象头

对象的结构：对象头+实例数据+补齐填充

对象头：MarkWord(动态数据结构)+ClassPoint(类型指针，指向方法区中类的元数据信息)

MarkWord在对象的不同状态下，数据结构不一样

- hashcode⇒无锁
- 线程ID⇒偏向锁
- 指向栈帧中锁记录⇒轻量级锁
    - 多个线程可在不同时刻获取到，只要不发生竞争就无事发生
- 指向monitor对象的指针⇒重量级锁
  - 
- GC

#### 锁升级

锁升级：(JDK6中对锁进行优化，JDK6之前默认重量级锁)，锁升级不可逆

无锁→偏向锁→轻量级锁→重量级锁

- 无锁→偏向锁：优先给上次获得锁的线程
- 偏向锁→轻量级锁
    - 只会被一个线程持有
    - 批量重偏向：多次都偏向成功会保持偏向
    - 批量撤销偏向：多次偏向都失败会锁升级
- 轻量级锁→重量级锁：cas retry 失败后认为有竞争，转为重量级锁
    - 通过CAS的方式尝试将栈帧中的锁记录指向markword
    - cas失败情况
        - 锁重入：不会锁升级，生成新栈帧，新锁记录作为计数
        - CAS失败：有竞争(轻量级锁绝对不允许竞争发生)，锁升级(锁膨胀)

  自适应自旋锁优化：以前获取不到锁，立马阻塞，加入阻塞队列，尝试获取锁。优化后，不会like放入阻塞队列，尝试十次cas操作（阻塞耗性能，java线程是内核线程模型，对应内核线程，阻塞涉及用户态到内核态切换），自适应主要解决忙轮询，根据上次锁的获取状态和释放锁时机决定重试次数

#### synchronized如何实现同步/互斥效果（重量级锁）



TODO

### 2、ReentrantLock

概述：Java层面实现的一把可重入锁，基于AQS，

AQS：Java中实现的锁、同步器的核心组件

```Java
public class ReentrantLockimplements Lock {
  private Sync sync;
  
  void lock(){
    sync.lock();
  }
  void unlock(){
    sync.release();
  }
  
  public class Sync extends AbstractQueueSynchronizer {
    @override...
  }
}

```



```Java
//抽象类AbstractOwnableSynchronizer内部维护了一个ownerThread(以获取到锁的线程)
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
  //多线程尝试获取锁时，先用CAS的方式修改state（不同实现下，state意义不同）
  volatile int state;
  //未获取到锁的线程，进入阻塞队列AQS通过LockSupport.park，下次被前置节点使用unlock唤醒
  //阻塞队列
  //获取到锁之后发现条件不满足进入条件队列。类似synchronized中的wait、notify等待唤醒机制，ReentryLock中是await、signal，
  //条件队列
}
```

总结：ReentrantLock就是实现了Lock接口，并提供了一些额外功能的API，但执行时，真正工作的事AQS，至于同步逻辑，是继承AQS后，重写方法自己定义的逻辑。

### 分类：

|||
|-|-|
|锁|应用|
|乐观锁|CAS|
|悲观锁||
|自旋锁||
|可重入锁||
|读写锁||
|公平锁||
|非公平锁||
|共享锁||
|独占锁||
|重量级锁||
|轻量级锁||
|偏向锁||
|分段锁||
|互斥锁||
|同步锁||
|死锁||
|锁粗化||
|锁消除||


