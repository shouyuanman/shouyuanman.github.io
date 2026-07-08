---
title: NioEventLoop——Netty实现多Reactor多线程模式
date: 2026-07-04 10:00:00 +0800
categories: [微服务, 高并发IO]
tags: [后端, 微服务, 高并发IO, NIO, Reactor, Netty]
music-id: 3373150034
---

## **引言**

从开发或者执行流程上，`Reactor`模式可以被清晰地分成三大步——注册、轮询、分发。
- 第一步，通道注册到选择器上；
- 第二步，反应器在选择器里进行`IO`事件轮询；
- 第三步，反应器把`IO`事件分发到处理器。

>`Netty`的`Reactor`模型，和经典`Reactor`模型的元素对应关系如下，
- `Netty`中的`Channel`，对经典的`Channel`做了封装；
- `Netty`的`Handler`，定义了自己`Handler`的规范；
- `Netty`的反应器，`NioEventLoop`。
{: .prompt-tip }

![Desktop View](/assets/images/20260627/netty_reactor.png){: width="800" height="400" }
_Netty的Reactor模型，和经典Reactor模型的元素对应关系_

### **EventLoop**

`Netty`中的`EventLoop`类，就是对应于`Reactor`反应器的`Reactor`反应器角色。

![Desktop View](/assets/images/20260627/netty_multi_thread_reactor.png){: width="500" height="250" }
_Netty中的Reactor模式示意图_

它是一个多线程的多反应器模式，
- 监听线程，`boss`线程，可以是多个；
- `IO`传输线程，反应器是`NioEventLoop`，多个`EventLoop`组成一个组，`workerLoopGroup`。

>要搞清楚`NioEventLoop`，首先需要搞清楚两个重要的关系，
- `NioEventLoop`与`NIO`原生`Selector`的关系
- `Netty Channel`与`NIO`原生`Channel`的关系
{: .prompt-tip }

`NioEventLoop`类型绑定了两大关键元素——`Thread`、`Java NIO Selector`，

![Desktop View](/assets/images/20260627/nioEventLoop_selector_thread.png){: width="400" height="200" }
_NioEventLoop的两大关键元素_

先看一下`Reactor`的`Java NIO Selector`选择器，

![Desktop View](/assets/images/20260627/nioEventLoop_class_code.png){: width="800" height="400" }
_NioEventLoop类定义_

再看下`Reactor`的线程`Thread`，

![Desktop View](/assets/images/20260627/singleThreadEventExecutor_class_code.png){: width="800" height="400" }
_NioEventLoop的Thread定义在父类SingleThreadEventExecutor中_

`NioEventLoop`的`Thread`
- `Thread`属性的作用，主要是用于轮询`NIO Selector`并处理`IO`事件；
- 处理其他的非`IO`任务。

>`NioEventLoop`的`Thread`线程什么时候启动呢？<br/>
在`Reactor`模式中，线程是轮询用的。所以，`Reactor`线程的启动，一般在`Channel`在`Selector`上注册之前，注册工作也是由`Thread`来完成的。
{: .prompt-tip }

>`EventLoop`和`Netty Channel`的关系<br/>
一对多的关系，一个`EventLoop`，可以注册很多不同的`Netty Channel`。
{: .prompt-tip }

![Desktop View](/assets/images/20260627/netty_channel_eventloop.png){: width="350" height="150" }
_`EventLoop`和`Netty Channel`，一对多_

选取`NioSocketChannel`类，作为`Channel`通道的代表，进行说明。

![Desktop View](/assets/images/20260627/nioSocketChannel_class.png){: width="500" height="250" }
_NioSocketChannel类的层次结构_

在`AbstractNioChannel`里，有一个`SelectableChannel`成员，这个成员是`Java JDK`原生`channel`成员，也就是说`Netty`的`Channel`封装了`JDK`原生`Channel`。

![Desktop View](/assets/images/20260627/abstractNioChannel_class_code.png){: width="800" height="400" }
_AbstractNioChannel类定义_

>回到正题，`Reactor`（对应`Netty`中`EventLoop`）三步曲
- 第一步：注册
    - 将`channel`通道的就绪事件，注册到选择器`Selector`。
    - 一个`Reactor`对应一个选择器`Selector`，一个`Reactor`拥有一个`Selector`成员属性。
- 第二步：轮询
    - 轮询的代码，是`Reactor`重要的一个组成部分，或者说核心的部分。轮询选择器是否有就绪事件。
- 第三步：分发
    - 将就绪事件，分发到事件附件的处理器`handler`中，由`handler`完成实际的处理。
{: .prompt-tip }

## **EventLoop三步曲**

### **第一步：注册**

在`Netty`中，注册流程还是比较复杂的，`Channel`注册选择器`Selector`调用流程如下，

```markdown
AbstractBootstrap.initAndRegister
--> MultithreadEventLoopGroup.register
--> SingleThreadEventLoop.register
--> AbstractChannel.AbstractUnsafe.register
--> AbstractChannel.register0
--> AbstractNioChannel.doRegister
```

注册的入口代码，在启动类`AbstractBootstrap`的`initAndRegister`方法中。

第一个步骤，是在引导类里边，`AbstractBootstrap.initAndRegister`，我们关注的是流程的最后一步，`AbstractNioChannel.doRegister`，拿到原生的选择器和原生的通道，完成通道的注册工作，

![Desktop View](/assets/images/20260627/abstractNioChannel_doRegister_code.png){: width="800" height="400" }
_AbstractNioChannel.doRegister_

>解释下上面`SocketChannel`注册到`eventLoop`的`selector`上的代码。<br/>
前面讲到，`AbstractNioChannel`通道类有一个本地`Java`通道成员`ch`，在`AbstractNioChannel`的构造函数中，被初始化。<br/>
`Netty`的`Channel`通过`javaChannel()`方法，取得了`Java`本地`Channel`。它返回的是一个`Java NIO SocketChannel`。<br/>
然后，将这个`SocketChannel`注册到与`eventLoop`关联的`selector`上了。<br/>
通过最后一步，`Netty`终于将这个`SocketChannel`注册到与`eventLoop`关联的`selector`上了。
{: .prompt-tip }

再看下经典的选择器注册，调用`selectionKey.attach(object)`，就是在`socketChannel`注册到选择器上返回选择键后，把`Acceptor`处理器`AcceptorHandler`绑定到选择键实例上。

```java
Reactor() throws IOException {
    //Reactor初始化
    selector = Selector.open();
    serverSocket = ServerSocketChannel.open();
    InetSocketAddress address = new InetSocketAddress(NioDemoConfig.SOCKET_SERVER_IP, NioDemoConfig.SOCKET_SERVER_PORT);
    //非阻塞
    serverSocket.configureBlocking(false);

    //分步处理,第一步,接收accept事件
    SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);

    // SelectionKey.OP_ACCEPT
    serverSocket.socket().bind(address);
    logger.info("服务端已经开始监听：" + address);

    // attach callback object, AcceptorHandler
    sk.attach(new AcceptorHandler());
}
```

`channel.register`的第三个参数，是可以设置`selectionKey`的附加对象的，和调用`selectionKey.attach(object)`的效果一样。

```java
SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT, new AcceptorHandler());
```

对比下`Netty`的方法，处理不太一样，这里选择键的附件，不是分两步做，而是放在了第三个参数里边，它把`NioSocketChannel`实例作为附件放到选择键上，这个附件在后面轮询和分发时会用到。

```java
selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
```

`doRegister()`所传递的第三个参数是`this`，它就是一个`NioSocketChannel`的实例。意思是，`SocketChannel`对象自身，以附加字段的方式添加到了`selectionKey`中，供事件就绪后使用。

### **第二步：轮询**

`Netty`的轮询也相对复杂一点。

>Q：反应器的线程在哪里？<br/>
A：`NioEventLoop`的父类`SingleThreadEventExecutor`中，有一个`Thread thread`属性，存储了一个本地`Java`线程。
{: .prompt-tip }

>Q：线程在哪里启动呢？<br/>
A：`SocketChannel`注册到`eventLoop`的`Selector`前。
{: .prompt-tip }

回顾下，通道注册的流程，最后一步是完成原生通道到原生选择器的注册工作，

倒数第三步，`AbstractChannel.AbstractUnsafe.register`，完成了线程启动，

![Desktop View](/assets/images/20260627/AbstractChannel_AbstractUnsafe_register.png){: width="800" height="400" }
_AbstractChannel.AbstractUnsafe.register注册时_

这里的`eventLoop.execute()`方法调用，就是启动`EventLoop`线程的入口。

`register`方法里边，有两种情况，
- 第一种情况，反应器线程是启动的，并且当前线程正好是反应器线程（`inEventLoop`），两个条件都满足的情况下，就直接执行注册工作（`register0`，就是最后一步的注册）。
- 第二种情况，也就是大多数时候，`EventLoop`线程往往没有启动，这时就得先启动线程，再注册。

进到任务提交方法，`execute`方法中调用`startThread()`启动线程，

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_execute.png){: width="800" height="400" }
_SingleThreadEventExecutor.execute_

线程启动之前，任务的添加，只要不是能立即执行的任务，都需要加入到队列里边，

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_addTask.png){: width="700" height="350" }
_将通道注册的任务，加到taskQueue_

回到线程的启动，`EventLoop`线程的启动，如果Reactor线程没启动，就在`startThread`里边，调用了`doStartThread`方法，启动线程，

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_startThread.png){: width="800" height="400" }
_SingleThreadEventExecutor.startThread_

`STATE_UPDATER`是`SingleThreadEventExecutor`内部维护的一个属性，它的作用是标识当前的`thread`的状态。在初始的时候，`state == ST_NOT_STARTED`，因此第一次调用`startThread`时，就会进入到`if`条件调用到`doStartThread()`。

这是`EventLoop`比较关键的一个方法，线程池，每执行一个任务都会创建一个线程，这个线程往往都是一个新线程，
首先，把当前线程作为反应器线程，之后，执行事件轮询的业务方法。

`executor`是反应器组的成员，类型是`ThreadPerTaskExecutor`，一个特殊的线程池，每执行一个任务，就创建一个新线程。

![Desktop View](/assets/images/20260627/MultithreadEventExecutorGroup_class_code.png){: width="800" height="400" }
_线程的创建来源于MultithreadEventExecutorGroup线程工厂_

>线程的数量，从哪里来的？<br/>
在最开始时，装配引导类时，有`boss group`、`worker group`，`boss group`线程数，指定`1`个连接监听的线程就够了；`worker group`线程数，事件处理，默认不传就是`CPU`核数，比如`8`核`CPU`就是`16`个线程。<br/>
有多少个线程数，就有多少个反应器，`newChild`就是创建一个`EventLoop`，对应的就是一个线程，线程创建的时机和`EventLoop`创建的时机不一样，线程是在`EventLoop`执行任务的时候创建的（`doStartThread`），创建完线程之后，`EventLoop`的核心方法`run`就跑起来了。
{: .prompt-tip }

![Desktop View](/assets/images/20260627/start_set_boss_worker_group.png){: width="800" height="400" }
_启动服务时，设置bossGroup、workerGroup_

`NioEventLoop.run()`方法，就是`IO`事件轮询的方法，是一个死循环，比如`Selector`发生了可读事件，就会被`run`方法轮询到，`run`方法还做了异步任务（非`IO`任务）的执行，`register0`（`Java`的`Channel`绑定到`Selector`）也是由`run`方法执行的，

```java
public final class NioEventLoop extends SingleThreadEventLoop {
    
    // ...
    private volatile int ioRatio = 50;
    // ...
    
    @Override
    protected void run() {
        for (;;) {
            try {
                // select 策略选择
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    // 非阻塞的select策略
                    case SelectStrategy.CONTINUE:
                        continue;
                    // 阻塞的select策略
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));

                        // 'wakenUp.compareAndSet(false, true)' is always evaluated
                        // before calling 'selector.wakeup()' to reduce the wake-up
                        // overhead. (Selector.wakeup() is an expensive operation.)
                        //
                        // However, there is a race condition in this approach.
                        // The race condition is triggered when 'wakenUp' is set to
                        // true too early.
                        //
                        // 'wakenUp' is set to true too early if:
                        // 1) Selector is waken up between 'wakenUp.set(false)' and
                        //    'selector.select(...)'. (BAD)
                        // 2) Selector is waken up between 'selector.select(...)' and
                        //    'if (wakenUp.get()) { ... }'. (OK)
                        //
                        // In the first case, 'wakenUp' is set to true and the
                        // following 'selector.select(...)' will wake up immediately.
                        // Until 'wakenUp' is set to false again in the next round,
                        // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                        // any attempt to wake up the Selector will fail, too, causing
                        // the following 'selector.select(...)' call to block
                        // unnecessarily.
                        //
                        // To fix this problem, we wake up the selector again if wakenUp
                        // is true immediately after selector.select(...).
                        // It is inefficient in that it wakes up the selector for both
                        // the first case (BAD - wake-up required) and the second case
                        // (OK - no wake-up required).

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    // 不需要select，目前已经有可执行的任务了
                    default:
                }

                // 执行网络IO事件和任务调度
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        // 处理网络IO事件
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        // 处理系统Task和自定义Task
                        runAllTasks();
                    }
                } else {
                    // 根据ioRatio计算非IO最多执行的时间
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}
```

`NioEventLoop`事件轮询包含两种策略，非阻塞的`select`策略、阻塞的`select`策略。

在这个死循环里边，`processSelectedKeys`就在分发`IO`事件，`runAllTasks`，就是处理任务队列的任务。

`NioEventLoop`中维护了一个线程，线程启动时会调用`NioEventLoop`的`run`方法，执行`IO`任务和非`IO`任务，
- `‌IO`任务‌：即`selectionKey`中`ready`的事件，如`accept`、`connect`、`read`、`write`等，由`processSelectedKeys`方法触发。
- 非`IO`任务‌：添加到`taskQueue`中的任务，如`register0`、`bind0`等任务，由`runAllTasks`方法触发。

两种任务的执行时间比由变量`ioRatio`控制，默认为`50`，则表示允许非`IO`任务执行的时间与`IO`任务的执行时间相等。

### **第三步：分发**

先看一下整个查询的过程，事件分发有个前提，`IO`事件已经通过选择器查询出来了。

`run`方法里死循环的第一步，就是事件的查询（把选择器里边的事件查出来），这个查询相较于经典`Reactor`模式有点复杂，但是原理类似，这里分多种不同策略，比如阻塞式查询、非阻塞查询，接下来就到了`processSelectedKeys`，做事件分发。

`processSelectedKeys`处理`IO`事件，这里有两种情况，第一种是优化过的，第二种是普通的。`Netty`对`Selector`做过优化，`Netty`会尝试获取权限去优化原生`Selector`，如果可以，`selectedKeys`不为`null`，就会走优化过的处理方式。优化不成功，就还是用原始经典的`Selector`选择器事件处理方式进行事件处理，这里先不做优化的展开介绍。两种方式主要是遍历`selectionKey`的方式不同，具体处理事件的调用逻辑是完全一致的。

![Desktop View](/assets/images/20260627/NioEventLoop_processSelectedKeys.png){: width="500" height="250" }
_NioEventLoop.processSelectedKeys方法实现_

下面介绍经典的选择键的处理，`processSelectedKeysPlain`，迭代`selectedKeys`获取就绪的`IO`事件，为每个事件都调用`processSelectedKey`来处理它。

![Desktop View](/assets/images/20260627/NioEventLoop_processSelectedKeysPlain.png){: width="600" height="300" }
_NioEventLoop.processSelectedKeysPlain方法实现_

在前面的`channel`注册时，将`NioSocketChannel`以附加字段的方式添加到了`selectionKey`中。在这里，通过`k.attachment()`取得这个通道对象，然后就调用`processSelectedKey`来处理这个`IO`事件和通道。

处理函数，会处理不同的事件，比如可写、可读，第一个参数是事件选择键，第二个是选择键的附件，比如对通道注册来说，附件就是通道，

![Desktop View](/assets/images/20260627/NioEventLoop_processSelectedKey.png){: width="800" height="400" }
_NioEventLoop.processSelectedKey方法实现_

`NioEventLoop.processSelectedKey`方法中处理了三个事件，
- OP_READ，可读事件，即Channel中收到了新数据可供上层读取。
- OP_WRITE，可写事件，即上层可以向Channel写入数据。
- OP_CONNECT，连接建立事件，即TCP连接已经建立，Channel处于active状态。

通道可读，会调用`NioByteUnsafe.read`方法，核心是`doReadBytes`，把通道里边的数据（发送过来的数据）读取出来，放到`ByteBuf`，

![Desktop View](/assets/images/20260627/AbstractNioByteChannel_NioByteUnsafe_read.png){: width="600" height="300" }
_NioByteUnsafe.read方法实现_

读取`NioSocketChannel`，`NioSocketChannel.doReadBytes`方法从`socket revbuf`读取数据，但每次读取前都需要记录缓冲区中可写区域的大小，用于判断缓冲区是否读满，继而决定是否继续读取数据。

![Desktop View](/assets/images/20260627/NioSocketChannel_doReadBytes.png){: width="800" height="400" }
_NioSocketChannel.doReadBytes方法实现_

`bytebuf.writeBytes`，`javaChannel`方法拿到`Java NIO`的原生通道，把通道的数据读取到`Bytebuf`里，读取的量是通过公式（可适配的）算出来的，比如按照之前监视的数据推断这次应该从通道里读多少数据。

把数据从通道里边读出来之后，下一步就是把读出来的数据分发出去。把`bytebuf`通过通道`channel`的流水线分发出去，分发给谁呢？就是流水线上面的处理器。流水线处理器就会读到`bytebuf`，`bytebuf`就这样一步步进入到业务处理器进行处理。

NioByteUnsafe.read 方法的后半部分，
- 触发`pipeline.fireChannelRead(byteBuf)`，通过`pipeline`触发读事件。
- 判断是否继续读有两个标准，一是不能超过最大的读取次数（默认`16`次）；二是缓冲区的数据每次都要读满，比如分配`2KB ByteBuf`，则必须读取`2KB`的数据。

整个事件的分发，就是这样的流程。`EventLoop`三步曲，就是这些了。

## **EventLoop无锁化设计**

`EventLoop`是一个`Reactor`模型的事件处理器，一个`EventLoop`对应一个线程（`EventLoop`只有一个线程），其内部会维护一个`selector`和`taskQueue`，负责处理网络`IO`请求和内部任务，这里的`selector`和`taskQueue`是线程内部的。

`selector`的`IO`事件处理，以及任务队列的任务处理，都在`EventLoop`线程串行的执行（内部的任务和事件的查询、处理、分发都是串行的，也就不存在线程安全问题，无锁化设计，性能非常高）。

## **NioEventLoop任务队列核心原理**

任务队列，对应的成员是`taskQueue`，位置在`SingleThreadEventExecutor`，这个队列比较特殊，不是普通的队列，在`newTaskQueue`里边创建的，创建的是`Mpsc`队列，实现由`JCTools`工具包提供，

![Desktop View](/assets/images/20260627/NioEventLoop_newTaskQueue.png){: width="700" height="350" }
_NioEventLoop.newTaskQueue方法实现_

>`Mpsc`，`Multi Producer Single Consumer` (`Lock less, bounded and unbounded`)，多生产者单一消费者无锁队列（有界和无界都有实现），来自`JCTools`工具包。<br/>
早在`1996`年就有论文提出了无锁队列的概念，再到后来`Disruptor`，高性能已得到生产的验证。`Jctools`中的高性能队列，其性能丝毫不输于`Disruptor`。
{: .prompt-tip }

`JCTools`（`Java Concurrency Tools`）提供了一系列非阻塞并发数据结构（标准`Java`中缺失的），当存在线程争抢的时候，非阻塞并发数据结构比阻塞并发数据结构能提供更好的性能`JCTools`是一个开源工具包，在`Apache License 2.0`下发布，并在`Netty`、`Rxjava`等诸多框架中被广泛使用。源码参见[`JCTools`的开源`Github`仓库](https://github.com/JCTools/JCTools)。

在`Maven`中引入`JCTools`坐标就能使用`JCTools`了，

```xml
<dependency>
    <groupId>org.jctools</groupId>
    <artifactId>jctools-core</artifactId>
    <version>3.0.0</version>
</dependency>
```

`JCTools`提供的数据结构，第一大类就是`Map`，`JCTools`提供的非阻塞`Map`，
- `ConcurrentAutoTable`（后面几个`map/set`结构的基础）
- `NonBlockingHashMap`（`ConcurrentHashMap`的增强）
- `NonBlockingHashMapLong`
- `NonBlockingHashSet`
- `NonBlockingIdentityHashMap`
- `NonBlockingSetInt`

另外，提供了非阻塞队列，`Netty`用到的是`MPSC`，`JCTools`提供的非阻塞队列分为`4`类，可以根据不同的应用场景选择使用，
- `SPSC`-单一生产者单一消费者（有界和无界）
- `MPSC`-多生产者单一消费者（有界和无界）
- `SPMC`-单生产者多消费者（有界）
- `MPMC`-多生产者多消费者（有界）

>任务队列的内容分为两部分，
- 任务的提交
- 任务的处理
{: .prompt-tip }

### **任务提交**

`Task`任务的提交，有`3`种典型使用场景，
- 用户提交的普通任务
- 用户提交的定时任务
- 非`Reactor`线程调用`Channel`的各种方法，例如在推送系统的业务线程里面，根据用户的标识，找到对应的`Channel`引用，然后调用`Write`类方法向该用户推送消息，就会进入到这种场景。最终的`Write`会提交到任务队列中后被异步消费。

普通任务提交，使用`eventLoop`的`execute`方法，通过`ChannelHandlerContext`获取`channel`，通过`channel`获取`eventLoop`，然后调用`execute`方法即可放入到任务队列，代码如下，

```java
Channel channel = ctx.channel();
channel.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
        try {
            Thread.sleep(1);
            // dosomething(...)
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});
```

例子就是通道注册时，

![Desktop View](/assets/images/20260627/AbstractChannel_AbstractUnsafe_register.png){: width="800" height="400" }
_AbstractChannel.AbstractUnsafe.register注册时_

定时任务提交，和普通任务类似，还是使用通道，`ctx.executor()`返回的是`EventLoop`，说明定时任务的提交，还是用的反应器`EventLoop`，只不过方法用的是`schedule`。

`Netty`提供了一些添加定时任务的接口，它就是`NioEventLoop`的父类`AbstractScheduledEventExecutor`的`schedule`方法，挑一个来看看（其它重载底层队列都一样），

定时任务也大同小异，都是通过`ChannelHandlerContext`获取`channel`，通过`channel`获取`eventLoop`，然后调用`schedule`方法即放入到任务队列，代码如下，

```java
//使用定时器，发送心跳报文
public void heartBeat(ChannelHandlerContext ctx,
                    ProtoMsg.Message heartbeatMsg) {
    ctx.executor().schedule(() -> {
        if (ctx.channel().isActive()) {
            log.info("发送 HEART_BEAT 消息 to server");
            ctx.writeAndFlush(heartbeatMsg);

            //递归调用，发送下一次的心跳
            heartBeat(ctx, heartbeatMsg);
        }
    }, HEARTBEAT_INTERVAL, TimeUnit.SECONDS);
}
```

`schedule`第一个参数和普通任务一样，传入一个线程即可，第二个参数是延时时间，第三个参数是延时单位，这里用的是秒。

`channel`的各种方法，是由`Reactor`线程执行的，非`Reactor`线程执行的话，就必须做一个任务，加入到任务队列，由`Reactor`线程后续调度执行。

```java
// 非反应器线程的消息发送操作
// 写Protobuf数据帧
public synchronized void writeAndFlush(Object pkg) {
    channel.writeAndFlush(pkg);
}
```

流程如下，如果`A`线程要调用`EventLoop channel`发消息，必须把发消息的操作变成一个任务，加入到`EventLoop`的`TaskQueue`，再由`EventLoop`里自己的线程从任务队列里边把任务取出来，再去执行消息的发送操作。

![Desktop View](/assets/images/20260627/non_reactor_thread_msg_send.png){: width="700" height="350" }
_非反应器线程的消息发送操作_

当用户线程（业务线程）发起`write`操作时，`Netty`会进行判断，如果发现不是`NioEventLoop`线程（反应器线程），则将发送消息封装成`WriteTask`，放入`NioEventLoop`的任务队列，由`NioEventLoop`线程后续去执行。

看一下过程大概对应的代码，用户线程（业务线程）发起`write`操作时的入口，

```java
io.netty.channel.AbstractChannelHandlerContext.write(java.lang.Object, boolean, io.netty.channel.ChannelPromise)
```

![Desktop View](/assets/images/20260627/AbstractChannelHandlerContext_writeAndFlush.png){: width="600" height="300" }
_AbstractChannelHandlerContext.writeAndFlush方法_

![Desktop View](/assets/images/20260627/AbstractChannelHandlerContext_write.png){: width="600" height="300" }
_AbstractChannelHandlerContext.write方法_

`write`里边，做一个判断，如果是`Reactor`线程自己在做（`inEventLoop`），直接干活；如果不是，就封装成一个任务，安全执行。整个过程和普通任务提交类似，只不过是定义了一个任务的类型，

![Desktop View](/assets/images/20260627/AbstractChannelHandlerContext_safeExecute.png){: width="800" height="400" }
_AbstractChannelHandlerContext.safeExecute方法_

这里的`executor`执行的是`Netty`自己实现的`SingleThreadEventExecutor.execute()`方法，这里把任务相关的参数封装了一下，普通的`Runnable`没有这么多成员。

为了节省内存，创建了一个可回收的对象池，`IO`通道有大量工作要做，并且很多都是异步的，不同的非`Reactor`线程发消息，任务实例需要复用。对象池就可以把这些实例回收，不需要每次都新建，可以反复使用，高效利用内存。

![Desktop View](/assets/images/20260627/AbstractChannelHandlerContext_AbstractWriteTask.png){: width="700" height="350" }
_AbstractChannelHandlerContext.AbstractWriteTask，WriteAndFlushTask的父类_

`execute`，任务提交方法，把传进来的`task`放到任务队列里边，不同`Netty`版本实现有些微小差异，

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_execute.png){: width="700" height="350" }
_SingleThreadEventExecutor.execute_

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_addTask.png){: width="600" height="300" }
_SingleThreadEventExecutor将通道注册的任务加到taskQueue队列_

至此，异步任务成功加入`taskQueue`。

`taskQueue`是`mpsc`队列，即多生产者单消费者队列，`Netty`使用`mpsc`，将外部线程的`task`聚集起来，在`Reactor`线程内部用单线程来无锁，串行执行。

### **任务调度**

异步任务的调度执行路径，代码调用路径如下，

```markdown
io.netty.channel.nio.NioEventLoop#run 
--> io.netty.util.concurrent.SingleThreadEventExecutor#runAllTasks(long) 
--> io.netty.util.concurrent.AbstractEventExecutor#safeExecute
```

这里`safeExecute`执行的`task`，就是前面`write`写入时包装的`AbstractWriteTask`，对应实现类路径为`io.netty.channel.AbstractChannelHandlerContext.AbstractWriteTask.run`。`AbstractWriteTask`的`run`经过一些系统处理操作，最终会调用`io.netty.channel.ChannelOutboundBuffer.addMessage`方法，将发送消息加入发送队列（链表）。

之前看过，查询到事件之后，就走到了`processSelectedKeys`，如果`IO`处理的比例是`100%`，就是所有查询出来的`IO`事件的活都要干，那就处理完所有`IO`事件，再处理所有任务队列里边的任务。

默认情况下，`IO`处理的比例是`50`，首先把时间记录下来，然后处理`IO`事件，得到处理`IO`事件的时间，根据比例计算任务处理的事件该多久。两边加起来的比例等于`1`，上面`IO`事件处理时间没办法压缩，可以控制的是任务处理时间。

看不加时间限制的`runAllTasks`方法，从定时任务里边，把到期了的任务取出来，把这些定时任务加到任务队列里边，

![Desktop View](/assets/images/20260627/NioEventLoop_runAllTasks.png){: width="800" height="400" }
_NioEventLoop.runAllTasks方法实现_

定时任务队列`scheduledTaskQueue`，是一个专门的任务队列，定义在`AbstractScheduledEventExecutor`，`EventLoop`的基类之一。

![Desktop View](/assets/images/20260627/AbstractScheduledEventExecutor_scheduledTaskQueue.png){: width="800" height="400" }
_AbstractScheduledEventExecutor.scheduledTaskQueue_

提交的定时任务，是提交到`ScheduledTaskQueue`里边了，只有它到期了，才把它取出来，加入到`TaskQueue`，直到把所有的定时任务都取出来。

```java
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    //从定时任务队列中抓取第一个定时任务
    //寻找截止时间为nanoTime的任务
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    //如果该定时任务队列不为空，则塞到普通任务队列里面
    while (scheduledTask != null) {
        //如果添加到普通任务队列过程中失败
        if (!taskQueue.offer(scheduledTask)) {
            //则重新添加到定时任务队列中
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
        //继续从定时任务队列中拉取任务
        //方法执行完成之后，所有符合运行条件的定时任务队列，都添加到了普通任务队列中
        scheduledTask  = pollScheduledTask(nanoTime);
    }
    return true;
}
```

`runAllTasksFrom`方法，执行任务队列里边所有的任务，

![Desktop View](/assets/images/20260627/SingleThreadEventExecutor_runAllTasksFrom.png){: width="800" height="400" }
_SingleThreadEventExecutor.runAllTasksFrom方法_

`safeExecute(task)`执行任务

![Desktop View](/assets/images/20260627/AbstractEventExecutor_safeExecute.png){: width="800" height="400" }
_AbstractEventExecutor.safeExecute方法_

举个例子，前面说的消息发送（前面`write`写入时），封装成了`AbstractWriteTask`，把要发送的消息、上下文都放到这个`task`实例里边，`AbstractWriteTask`的`run`只执行最终的通道写出，最终会调用到`addMessage`，将发送的消息加入到发送列表，最终发送出去。任务的执行都在`run`方法里边完成。

具体工作和通道的消息发送流程有关，有需要的话，后续做专题介绍。

## **ChannelConfig通道配置类**

>首先看，内存分配器怎么作用到通道上面的呢？<br/>
通过通道配置选项
{: .prompt-tip }

![Desktop View](/assets/images/20260627/channel_config_case.png){: width="700" height="350" }
_ChannelConfig通道配置类示例_

下面看下通道配置类和通道配置选项。

>什么是`ChannelConfig`？<br/>
`ChannelConfig`通道配置类用于管理各种通道的配置选项。每个特定的`Channel`实现类都有自己对应的`ChannelConfig`实现类，比如，
- `NioSocketChannel`对应的配置类为`NioSocketChannelConfig`；
- `NioServerSocketChannel`对应的配置类为`NioServerSocketChannelConfig`。\
{: .prompt-tip }

`Netty`的通道非常多，每种通道都有各自通道配置选项，专门设计了通道配置类来干这个事情。

![Desktop View](/assets/images/20260627/ChannelConfig_inheritance_relation.png){: width="600" height="300" }
_ChannelConfig的部分继承关系_

以`SocketChannel`通道配置类为例，
- 传输通道配置类
- 监听通道配置类
- 顶层接口，`ChannelConfig`
- 有一个默认实现配置类，`DefaultChannelConfig`

>如何通过`Channel`接口获取`ChannelConfig`的实例？<br/>
在`Channel`接口中定义了一个方法`config()`，用于获取特定通道实现的配置，`Channel`子类需要实现这个接口。
{: .prompt-tip }

```java
public interface Channel extends AttributeMap, Comparable<Channel> {
    ...
    ChannelConfig config();
    ...
}
```

>`ChannelConfig`实例的创建时机<br/>
通常`Channel`实例，在创建的时候，就会创建其对应的`ChannelConfig`实例。
{: .prompt-tip }

在构造函数里边，例如，`NioServerSocketChannel`和`NioSocketChannel`都是在构造方法中创建了其对应的`ChannelConfig`实现。

```java
public class NioServerSocketChannel extends AbstractNioMessageChannel
        implements io.netty.channel.socket.ServerSocketChannel {

    private final ServerSocketChannelConfig config;

    public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        // 构造方法中创建NioServerSocketChannelConfig实例
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
}
```
{: file='NioServerSocketChannel.java' .nolineno }

```java
public class NioSocketChannel extends AbstractNioByteChannel implements io.netty.channel.socket.SocketChannel
// ...
private final SocketChannelConfig config;
// ...
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    // 构造方法中创建SocketChannelConfig实例
    config = new NioSocketChannelConfig(this, socket.socket());
}
// ...
```
{: file='NioSocketChannel.java' .nolineno }

### **ChannelConfig的配置项ChannelOption**

`Netty`设计一个`ChannelOption`类，用于封装`ChannelConfig`配置项支持的所有选项。

![Desktop View](/assets/images/20260627/ChannelOption_inheritance_relation.png){: width="300" height="150" }
_ChannelOption的部分继承关系_

>`ChannelConfig`与`ChannelOption`的关系是什么呢？<br/>
`ChannelConfig`类似`Map`，`ChannelOption`是`Map`的`key`，`ChannelConfig`定义了相关方法来获取和修改`Map`中的值。
{: .prompt-tip }

```java
public interface ChannelConfig {
    //获取所有参数
    Map<ChannelOption<?>, Object> getOptions();

    //设置所有参数
    boolean setOptions(Map<ChannelOption<?>, ?> options);

    //获取以某个ChannelOption为key的参数值
    <T> T getOption(ChannelOption<T> option);

    //替换某个ChannelOption为key的参数值
    <T> boolean setOption(ChannelOption<T> option, T value);
}
```

举个例子，`ChannelConfig`类的选项修改。当我们想修改一个`Map`中的参数时，使用`setOption`方法。

例如，希望为`NioSocketChannel`的内存分配器配置为`PooledByteBufAllocator`，则可以使用类似以下方式来设置，

```java
Channel ch = ...;
SocketChannelConfig cfg = (SocketChannelConfig) ch.getConfig();
cfg.setOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

`Netty`预定了很多配置项，`ChannelConfig`支持的通用`ChannelOption`
- `ChannelOption.CONNECT_TIMEOUT_MILLIS` 连接超时设置
- `ChannelOption.WRITE_SPIN_COUNT`
- `ChannelOption.AUTO_READ`
- `ChannelOption.MAX_MESSAGES_PER_READ`
- `ChannelOption.RCVBUF_ALLOCATOR`
- `ChannelOption.ALLOCATOR` 内存分配器
- `ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK`
- `ChannelOption.WRITE_BUFFER_LOW_WATER_MARK`
- `ChannelOption.MESSAGE_SIZE_ESTIMATOR`
- `ChannelOption.AUTO_CLOSE`

和`socket`的传输设计有关系，`SocketChannelConfig`在`ChannelConfig`基础上额外支持的`ChannelOption`
- `ChannelOption.SO_KEEPALIVE`
- `ChannelOption.SO_REUSEADDR`
- `ChannelOption.SO_LINGER`
- `ChannelOption.TCP_NODELAY`
- `ChannelOption.SO_RCVBUF`
- `ChannelOption.SO_SNDBUF`
- `ChannelOption.IP_TOS`
- `ChannelOption.ALLOW_HALF_CLOSURE`

`ChannelOption`的对应`set/get`方法，事实上这里的每一种`ChannelOption`，除了可以使用`setOption`方法来进行设置，

```java
ch.setOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

在`ChannelConfig`接口中都为其设置了对应的快捷`set/get`方法。

```java
Channel ch = ...;
SocketChannelConfig cfg = (SocketChannelConfig) ch.getConfig();
cfg.setAllocator(PooledByteBufAllocator.DEFAULT);
```
