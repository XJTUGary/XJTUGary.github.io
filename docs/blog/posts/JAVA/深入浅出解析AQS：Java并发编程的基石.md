---
date: 2025-09-06
title: 深入浅出解析AQS：Java并发编程的基石
categories:
  - JAVA
  - 并发
draft: false
comments: true
---
在Java并发编程中，AbstractQueuedSynchronizer（AQS）是`java.util.concurrent.locks`包的核心基础，像ReentrantLock、Semaphore、CountDownLatch等常用同步工具都构建于其上。理解AQS，就掌握了JUC包的精髓。本文将从设计思想、核心机制到源码实现，带你彻底搞懂AQS。
<!-- more -->


## 一、AQS解决了什么问题？

AQS的核心目标是：**当多个线程竞争共享资源时，如何高效、安全地管理线程的排队与唤醒**。它通过三大组件解决这个问题：

1.  **volatile int state**：表示共享资源的状态（如锁的可重入次数、信号量的许可证数）
2.  **FIFO双向队列**：封装竞争失败的线程，让它们排队等待
3.  **模板方法模式**：AQS提供核心框架，子类定义具体资源分配规则

## 二、状态变量state的设计奥秘

为什么state是int类型而不是boolean？

```java
    /**
     * The synchronization state.
     */
    private volatile int state;
```

因为需要支持更丰富的并发场景：
- **互斥场景**：state=0表示空闲，1表示被占用（ReentrantLock）
- **共享场景**：state表示可用资源数（如Semaphore允许多个线程同时访问）
- **可重入场景**：state记录重入次数，重入一次加1，释放一次减1

通过`volatile`保证可见性，通过CAS操作保证原子性，为并发控制提供基础。
```java
    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
    protected final boolean compareAndSetState(int expect, int update) {
        return U.compareAndSetInt(this, STATE, expect, update);
    }
```

## 三、等待队列的精妙设计

当线程通过CAS修改state失败后，AQS如何管理这些等待线程？

### 1. Node节点结构

每个等待线程被封装成Node节点：

```java
static final class Node {
    volatile int waitStatus; // 等待状态：CANCELLED、SIGNAL、CONDITION等
    volatile Node prev;      // 前驱节点
    volatile Node next;      // 后继节点
    volatile Thread thread;  // 线程引用
    Node nextWaiter;         // 条件队列链接
}
```

### 2. 为什么选择双向链表？

AQS使用双向链表实现FIFO队列，这种设计有两大优势：
- **插入删除高效**：O(1)时间复杂度
- **支持公平策略**：队列的先进先出特性为实现公平锁奠定基础

在“等待队列的精妙设计”中我们提到，AQS使用了一个FIFO双向队列。一个很自然的问题是：为什么是**双向**链表？相比单向链表，每个节点需要额外维护一个`prev`前驱指针，这似乎带来了额外的内存开销。然而，这个设计是AQS能够高效、可靠处理复杂并发场景的基石，其核心优势在于**支持高效地删除任意节点**。

#### 1. 高效处理取消操作（Cancellation）

在多线程环境中，一个正在队列中等待获取锁的线程可能会因为**超时（`tryAcquireNanos`）** 或**被中断（`acquireInterruptibly`）** 而需要取消等待，将其对应的节点从队列中移除。

*   **单向链表的困境**：如果一个节点（Node B）需要取消，要移除它，必须知道它的前驱节点（Node A）。但在单向链表中，Node B只持有后继节点Node C的引用。要找到Node A，**必须从链表头部开始遍历**，直到某个节点的`next`指向Node B。这个遍历操作的时间复杂度是O(n)，在高并发场景下，长队列会带来巨大的性能开销。

*   **双向链表的优势**：Node B本身就保存了指向其前驱节点（Node A）的引用（`node.prev`）。移除Node B变得非常简单：
    1.  通过`B.prev`直接找到Node A。
    2.  将`A.next`指向`B.next`。
    3.  如果`B.next`（Node C）不为空，将`C.prev`指向Node A。

这个过程**无需遍历**，时间复杂度是O(1)，极大地提升了性能。

**源码佐证**：在 `acquireQueued` 方法的 `finally` 块中，如果获取资源失败（例如发生异常），会调用 `cancelAcquire(node)` 方法。这个方法内部大量依赖了 `node.prev` 引用来进行复杂的链表清理和重建工作，这正是双向链表价值最直接的体现。

#### 2. 显式维护前驱节点的状态（SIGNAL）

在AQS中，一个节点的线程在阻塞自己（`park`）之前，需要确保它的前驱节点的`waitStatus`是`SIGNAL`（`shouldParkAfterFailedAcquire`方法）。这建立了一种“责任链”协议：**“我（前驱节点）释放资源后会负责唤醒你（当前节点），所以你现在可以安心睡觉了”**。

如果一个节点的前驱节点取消了（`waitStatus = CANCELLED`），它需要**跳过**这些已取消的节点，找到一个有效的、状态为`SIGNAL`的前驱节点，并重新建立链接。双向链表使得节点可以轻松地**向前遍历**（通过`prev`指针），高效地完成这个“寻找有效前驱”的任务。

**源码佐证：`shouldParkAfterFailedAcquire` 方法**
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) { // 前驱节点已取消（CANCELLED）
        // 关键：通过prev指针向前遍历，跳过所有已取消的节点
        do {
            node.prev = pred = pred.prev; // 双向链表的核心！
        } while (pred.waitStatus > 0);
        pred.next = node; // 重新链接链表
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
如果没有`prev`指针，这个“跳过已取消节点”的操作将变得极其低效，需要从队列头开始遍历。

#### 3. 条件队列（Condition Queue）功能的支撑

AQS的强大功能之一是其条件变量（`ConditionObject`），而这严重依赖双向链表的结构。

*   当调用 `condition.await()` 时，当前线程会**释放锁**，并将当前节点从**同步队列**（锁队列）转移到**条件队列**（一个由`nextWaiter`连接的单向链表）。
*   当调用 `condition.signal()` 时，需要将节点从条件队列**转移回同步队列**。

这个“转移”操作涉及到将节点重新插入到同步队列的尾部。而一旦节点回到同步队列，它就可能会面临上述的**取消**和**状态维护**问题。因此，一个必须是双向链表的同步队列，是高效实现条件变量机制的基础。


## 四、核心流程源码解析

### 1. 获取资源 - tryAcquire/acquire流程

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 子类实现的具体获取逻辑
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

这是一个经典模板方法模式，流程如下：

1.  **tryAcquire**：子类定义资源获取规则（如ReentrantLock中尝试加锁）
2.  **addWaiter**：将线程包装为Node并加入队列尾部
3.  **acquireQueued**：在队列中循环尝试获取资源，必要时阻塞线程

### 2. 释放资源 - release流程

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) { // 子类实现的具体释放逻辑
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒后继节点
        return true;
    }
    return false;
}
```

释放资源后，AQS会唤醒队列中合适的等待线程，使其重新尝试获取资源。

## 五、AQS的两种模式

### 1. 独占模式（Exclusive）

一次只有一个线程能访问资源，适用于互斥场景，如ReentrantLock。

- **获取**：acquire、tryAcquire
- **释放**：release、tryRelease

### 2. 共享模式（Shared）

多个线程可以共享资源，适用于协作场景，如Semaphore、CountDownLatch。

- **获取**：acquireShared、tryAcquireShared
- **释放**：releaseShared、tryReleaseShared

## 六、AQS高级特性：Condition条件队列

除了同步队列，AQS还通过ConditionObject支持条件队列：

```java
public class ConditionObject implements Condition {
    private transient Node firstWaiter; // 条件队列头
    private transient Node lastWaiter;  // 条件队列尾
}
```

这与同步队列的区别在于：
- **同步队列**：存放等待锁的线程
- **条件队列**：存放调用了await()主动放弃锁的线程

通过多个Condition对象，可以实现精细化的线程通信和唤醒。

## 七、为什么不用操作系统互斥锁？

AQS在用户态实现同步机制有显著优势：

1.  **性能高效**：避免用户态-内核态切换的开销
2.  **灵活可扩展**：Java层面可自定义公平策略、共享模式等
3.  **功能丰富**：支持可重入、中断、超时、条件等待等复杂特性

## 八、AQS在实际开发中的应用

### 1. ReentrantLock vs synchronized

基于AQS的ReentrantLock提供了更多高级功能：

| 特性 | ReentrantLock | synchronized |
|------|---------------|-------------|
| 尝试非阻塞获取 | ✅ tryLock() | ❌ |
| 可中断 | ✅ lockInterruptibly() | ❌ |
| 公平锁 | ✅ 支持 | ❌ |
| 条件队列 | ✅ 多条件 | ❌ 单条件 |

### 2. 常用同步工具实现原理

- **Semaphore**：state表示许可证数量，acquire获取许可证，release释放许可证
- **CountDownLatch**：state表示剩余任务数，countDown递减，await等待归零
- **ReentrantReadWriteLock**：通过两个AQS子类分别管理读锁和写锁

## 九、总结

AQS是Java并发包的基石，通过state状态管理、CLH队列和模板方法模式，提供了一个高度灵活、高效的同步框架。理解AQS的设计思想和实现原理，不仅有助于我们更好地使用JUC包中的同步工具，也为设计和实现自定义同步器提供了坚实基础。