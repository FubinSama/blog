# AQS

## AbstractOwnableSynchronizer抽象类

该接口用来标志同步器当前持有排他锁的线程

```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    private static final long serialVersionUID = 3737899427754241961L;
    protected AbstractOwnableSynchronizer() { }
    private transient Thread exclusiveOwnerThread; // 代表当前持有排他锁的线程
    protected final void setExclusiveOwnerThread(Thread thread) { exclusiveOwnerThread = thread; }
    protected final Thread getExclusiveOwnerThread() { return exclusiveOwnerThread; }
}
```

## JUC和AQS简介

JUC包提供了一系列的中层的并发支持类。这些类是一系列的并发同步器。而并发同步器是一种抽象数据结构（ADT），它维持着内部同步状态（比如说：锁的lock和unlock）的，可以操作和获取状态，并且至少有一种方法可以在有需要时让调用的线程阻塞，然后在其他线程更改同步状态以允许它继续执行时恢复。

同步器包括：各种互斥锁实现，读写锁，信号量（semaphores），屏障（barriers），futures，事件指示器和handoff队列等。

众所周知，同步器只要实现了一种，就可以用它来实现其他的全部。但是，这样的实现通常会带来足够的复杂性、开销和不灵活性，充其量只是二流的工程选择。
此外，它在概念上没有吸引力。如果这些结构中没有一个本质上比其他结构更原始，开发人员不应被迫任意选择其中一个作为构建其他结构的基础。
相反，JSR166 建立了一个以类 AbstractQueuedSynchronizer 为中心的小型框架，它提供了大多数同步器使用的通用机制，它拥有两种方法：至少一种获取操作会阻塞调用线程，直到同步状态允许它继续进行，并且至少有一个释放操作以一种可能允许一个或多个阻塞线程解除阻塞的方式更改同步状态。

JUC包没有为同步器定义一个简单通用的API。虽然它也定义了一些通用的接口（如：Lock），但很多同步器会有特别的版本。所以，`acquire`和`release`操作在不同的类中可能会有不同的名字，如：`Lock.lock`，`Semaphore.acquire`，`CountDownLatch.await`和`FutureTask.get`方法实际上都是`acquire`操作。

不过，JUC包在类支持的操作上维持了一致，每个同步器都支持：

1. 尝试非阻塞同步（如：tryLock）和阻塞同步（如：lock）。
2. 可选的超时。因此，应用可以放弃等待
3. 通过中断（interrupt）取消，通常分为一个可取消的`acquire`版本和不可取消的版本。

同步器可能仅管理独占状态（其中一次只有一个线程可能会继续通过可能的阻塞点），也可能要支持共享状态（有时有多个线程可以继续通过阻塞点）。常规锁类当然只维护独占状态，但是计数信号量（只要计数允许）可能会被多个线程获取。要广泛使用，框架就必须同时支持这两种操作模式。

JUC包还定义了Condition接口，用来支持监视器样式的wait/signal操作，这些操作可能与Lock类相关联，并且其实现本质上与其关联的Lock类交织在一起。

## 设计和实现

同步器背后的基本思想非常简单。

`acquire`操作流程如下:

```java
while (synchronization state does not allow acquire) { 
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;
```

`release`操作流程如下：

```java
update synchronization state;
if (state may permit a blocked thread to acquire) {
    unblock one or more queued threads;
}
```

支持这些操作需要协调三个基本组件：

1. 自动化管理同步状态
2. block和unblock当前线程
3. 维护队列

同步器框架的核心设计决策是选择这三个组件中每一个的具体实现，同时仍然允许在如何使用它们方面有广泛的选择。这有意限制了适用范围，但提供了足够有效的支持，以至于在适用的情况下几乎没有理由不使用该框架。

### 同步状态

类`AbstractQueuedSynchronizer`维持同步状态只使用了一个`int`类型的变量`state`，并暴露了`getState`、`setState`和`compareAndSetState`操作去访问和更新同步状态。

这些方法反过来依赖于`java.util.concurrent.atomic`包提供的`JSR133（Java内存模型）`兼容的读写`volatile`语义的支持（实际上`JDK`收录的`AQS`对于`state`变量直接使用的`volatile`关键字）。并通过访问`native`的`compare-and-swap`或`load-linked/store-conditional`指令以实现比较`compareAndSetState`操作，该操作只有当状态保持给定的预期值时，它才会自动将状态设置为给定的新值。

将同步状态限制为32位`int`是一个务实的决定。虽然`JSR166`还提供了对64位长字段的原子操作，但实际上在很多的平台上它必须使用内部锁来模拟这些操作，这样生成的同步器同步器性能不佳。将来，可能会添加第二个基类，该基类专门用于64位状态。但是，现在还没有必要（这里说的是`AbstractQueuedLongSynchronizer`，JDK5的确没有，但JDK6就有了）。因为目前32位足以满足大多数应用程序的需求。可能`CyclicBarrier`需要更多位来维护状态，所以该类使用`Lock`而不是通过`AQS`来实现。

基于`AbstractQueuedSynchronizer`的具体类必须根据这些导出`state`的方法来定义`tryAcquire`和`tryRelease`方法，以控制同步的获取和释放操作。如果已获取同步，则`tryAcquire`方法必须返回`true`，如果新的同步状态可能允许将来获取同步，则`tryRelease`方法必须返回`true`。这些方法接受单个`int`参数，可用于传达期望释放的状态，例如：在可重入锁中，在等待的条件就绪以后重新获取锁时需要重新构建递归计数（这里的意思是：可重入锁，就是释放几层递归计数，信号量就是释放多少个资源）。许多同步器不需要这样的参数，直接忽略它就是了。

### 阻塞

在`JSR166`之前，没有可用的`Java API`来block和unblock线程以创建不基于内置监视器的同步器。仅有的候选者`Thread.suspend`和`Thread.resume`也是不可用的，因为它们会引入不可解决的竞争问题：如果一个非阻塞的线程在要阻塞的线程执行`suspend`前执行了`resume`，那么`resume`操作就无效了。
`java.util.concurrent.locks`包引入了`LockSupport`类来解决这个问题。方法`LockSupport.park`阻塞当前线程，除非或直到发出`LockSupport.unpark`（也允许虚假唤醒）。`unpark`的调用是不计数的，所以在一个`park`之前多次调用`unpark`也只会unblock一个`park`。此外，这适用于每个线程，而不是每个同步器。一个线程在一个新的同步器上执行`park`可能会立即返回，因为可能有线程之前已经调用过`unpark`而且没被消耗。然而，在没有`unpark`的情况下，它的下一次调用将被阻塞。虽然可以显式清除此状态，但不值得这样做。必要时多次调用`park`会更有效。

这种简单的机制在某种程度上类似于 Solaris-9 线程库、WIN32“消费事件”和 Linux NPTL 线程库中使用的机制，因此可以有效地映射到Java运行的通用平台。 （但是，当前在 Solaris 和 Linux 上的 Sun Hotspot JVM 参考实现实际上使用 pthread condvar 以适应现有的运行时设计。）park 方法还支持可选的相对和绝对超时，并集成了JVM的Thread.interrupt的支持——interrupt一个线程会将其unpark。

### 队列
