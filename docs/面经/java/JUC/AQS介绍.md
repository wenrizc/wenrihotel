
AQS为实现依赖于先进先出（FIFO）等待队列的阻塞锁和相关的同步器（如信号量、事件等）提供了一个基础框架。 像ReentrantLock、Semaphore、CountDownLatch等并发工具，都是基于AQS来构建的。 AQS的设计目标是简化和高效地构建锁和同步器。

#### AQS的核心思想

AQS的核心思想在于，当一个线程请求共享资源时：
*   **如果资源空闲**：则将该线程设置为有效的工作线程，并将共享资源置为锁定状态。
*   **如果资源已被占用**：那么就需要一套机制来管理后续的请求线程。AQS采用的是一个CLH队列锁的变体来实现，将暂时无法获取到锁的线程加入到一个FIFO队列中进行等待。 当锁被释放时，再从队列中唤醒一个线程来获取锁。

### AQS的实现原理

AQS的实现主要围绕着以下几个核心组件和机制展开：

#### 1. 同步状态 (state)

AQS内部维护了一个`volatile`的整型变量`state`，用来表示同步状态。
*   `state`为0时，通常表示资源未被锁定。
*   `state`大于0时，表示资源已被锁定。对于可重入锁来说，这个值还可以表示锁被同一个线程重入的次数。

AQS提供了三个核心方法来原子性地操作`state`的值，这三个方法内部利用了CAS（Compare-And-Swap）操作来保证线程安全：
*   `getState()`: 获取当前同步状态的值。
*   `setState(int newState)`: 设置同步状态的值。
*   `compareAndSetState(int expect, int update)`: 如果当前`state`的值等于`expect`，则原子性地将其更新为`update`。

#### 2. CLH队列 (Craig, Landin, and Hagersten Lock)

AQS内部实现了一个CLH队列的变体，这是一个虚拟的双向队列，用于存放等待获取锁的线程。 每个请求共享资源的线程都会被封装成队列中的一个节点（Node）。

*   **节点 (Node)**：每个Node节点包含了线程本身、前驱节点（prev）、后继节点（next）以及等待状态（waitStatus）等信息。通过`prev`和`next`指针，节点之间构成了一个双向链表。
*   **入队与出队**：当线程获取锁失败后，会被封装成一个Node节点并加入到队列的尾部。当持有锁的线程释放锁时，会唤醒队列头部的下一个节点所对应的线程。

AQS对传统的CLH自旋锁进行了优化，引入了阻塞和唤醒机制。当线程获取锁失败后，会先短暂自旋尝试，如果仍然失败，则会被挂起（park），进入阻塞状态，从而减少不必要的CPU资源消耗。

#### 3. 两种资源共享模式

AQS定义了两种不同的资源共享方式，以满足不同场景下的并发需求：

*   **独占模式 (Exclusive)**：在任何时刻，只允许一个线程持有锁，例如`ReentrantLock`。
*   **共享模式 (Share)**：允许多个线程同时获取锁，例如`Semaphore`和`CountDownLatch`。

#### 4. 模板方法设计模式

AQS的设计采用了模板方法模式。 它将通用的线程排队、阻塞、唤醒等逻辑封装在自身内部，而将与具体锁实现相关的逻辑（即如何获取和释放同步状态`state`）开放给子类去实现。

开发者如果需要自定义同步器，只需继承AQS并重写以下几个`protected`方法即可：

*   `tryAcquire(int arg)`：独占模式下尝试获取资源。
*   `tryRelease(int arg)`：独占模式下尝试释放资源。
*   `tryAcquireShared(int arg)`：共享模式下尝试获取资源。
*   `tryReleaseShared(int arg)`：共享模式下尝试释放资源。
*   `isHeldExclusively()`：判断当前线程是否持有独占锁。

以`ReentrantLock`为例，它的`lock`和`unlock`方法内部都聚合了一个继承自AQS的`Sync`子类（分为公平锁`FairSync`和非公平锁`NonfairSync`）。 当调用`lock()`方法时，实际上是调用了AQS的`acquire()`方法，`acquire()`会再回调子类重写的`tryAcquire()`方法来真正执行获取锁的逻辑。 如果获取失败，AQS框架就会接管后续的入队和线程挂起等操作。 同样，`unlock()`方法会调用AQS的`release()`方法，该方法则会回调子类的`tryRelease()`来释放锁，并由AQS完成后续的唤醒操作。