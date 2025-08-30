
AQS，全称是AbstractQueuedSynchronizer，它是Java并发包（`java.util.concurrent`，即JUC）中一个非常核心的抽象类。可以把它理解为是一个用来构建锁和其他同步组件的底层基础框架。像我们熟知的`ReentrantLock`、`Semaphore`、`CountDownLatch`等，它们的内部实现都离不开AQS。

AQS的设计目标，就是把实现一个同步器时通用且繁琐的部分给封装起来，比如管理等待线程的队列、线程的阻塞与唤醒等。这样，开发者只需要关注并实现与业务逻辑相关的、最核心的同步状态管理即可。

我将从它的设计思想、核心组件和工作流程这三个方面来详细介绍AQS。

### 核心设计思想：模板方法模式

AQS的设计巧妙地运用了“模板方法”这一设计模式。

- AQS作为父类，定义好了同步器实现的主体逻辑框架（即模板）。这个框架规定了线程如何尝试获取资源、获取失败后如何排队、以及资源释放后如何通知后续线程等一系列标准流程。
    
- 子类通过继承AQS，并重写它提供的几个受保护的（protected）方法，来定义自己同步器的特定规则。这些需要被重写的方法，主要就是对同步状态（state）的获取和释放操作。
    

通过这种方式，AQS将通用的队列管理和线程调度逻辑与具体的资源状态同步逻辑解耦开来，极大地降低了开发一个可靠、高效的同步器的难度。

### 两个核心的内部组件

AQS的内部实现主要依赖于两个关键部分：

1. 一个volatile的int类型变量：state
    
    这个state变量是AQS的核心，它代表了同步资源的状态。volatile关键字保证了它在多线程之间的可见性。子类需要根据自己的逻辑来定义state的含义，并通过AQS提供的getState()、setState()以及compareAndSetState()（CAS操作）这几个方法来安全地读写这个状态值。
    
    举几个例子：
    
    - 在`ReentrantLock`中，`state`表示锁的持有状态。`state=0`表示未被锁定，`state>0`表示已被锁定，其值代表了锁的重入次数。
        
    - 在`CountDownLatch`中，`state`表示还需要倒数的计数量。
        
    - 在`Semaphore`中，`state`表示当前剩余的许可数量。
        
2. 一个FIFO双向链表：CLH队列
    
    这个队列用来存放所有请求资源失败而被阻塞的线程。当一个线程尝试获取同步状态失败后，AQS会将其封装成一个Node节点，并以原子方式（CAS）加入到这个队列的尾部。随后，该线程会被挂起（park），进入休眠状态，等待被唤醒。
    
    这个队列是AQS实现线程排队、公平锁和非公平锁等机制的基础。
    

### 主要工作流程：Acquire（获取）与Release（释放）

AQS的工作流程可以分为两种模式：独占模式（Exclusive）和共享模式（Shared）。我们以最常见的独占模式（如`ReentrantLock`）为例，来讲解其核心的获取和释放流程。

**1. 获取流程 (Acquire)**

1. 线程调用`lock.acquire()`（或`lock.lock()`）尝试获取锁。
    
2. `acquire()`方法会首先调用子类实现的`tryAcquire()`方法。这是一个关键的自定义点。
    
3. 如果`tryAcquire()`返回`true`：代表线程成功获取了锁，方法直接返回，线程继续执行业务逻辑。
    
4. 如果tryAcquire()返回false：代表获取锁失败。此时AQS的框架逻辑开始工作：
    
    a. 将当前线程封装成一个Node对象。
    
    b. 通过CAS操作，将这个Node节点追加到CLH等待队列的尾部。
    
    c. 将线程挂起，使用LockSupport.park()让其进入等待状态，防止它消耗CPU。
    

**2. 释放流程 (Release)**

1. 持有锁的线程调用`lock.release()`（或`lock.unlock()`）来释放锁。
    
2. `release()`方法会调用子类实现的`tryRelease()`方法。
    
3. 在`tryRelease()`中，子类会去更新`state`变量（比如在`ReentrantLock`中将`state`减1）。
    
4. 如果tryRelease()返回true：代表锁被完全释放了（比如ReentrantLock的state变为了0）。此时AQS框架会：
    
    a. 找到CLH队列的头节点。
    
    b. 唤醒（unpark）头节点中包装的那个等待线程。
    
5. 被唤醒的线程会再次尝试执行`tryAcquire()`来获取锁。如果成功，它就成为新的锁持有者；如果失败（例如在非公平锁模式下被插队），它会继续留在队列中等待。
    
