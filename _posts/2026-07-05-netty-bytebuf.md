---
title: ByteBuf——Netty定义的NIO ByteBuffer升级组件
date: 2026-07-05 10:00:00 +0800
categories: [微服务, 高并发IO]
tags: [后端, 微服务, NIO, Reactor, Netty]
music-id: 3373150034
---

## **引言**

`NIO`的`Buffer`本质是一个内存块，`NIO`的`Buffer`类是一个位于`java.nio`包的抽象类，它有`8`种缓冲区子类，分别是`ByteBuffer`、`CharBuffer`、`DoubleBuffer`、`FloatBuffer`、`IntBuffer`、`LongBuffer`、`ShortBuffer`、`MapperByteBuffer`，其中使用最多的就是`ByteBuffer`二进制字节缓冲区类型。

这些子类会拥有一块内存，作为数据的读写缓冲区，比如`ByteBuffer`就有一个`byte[]`类型的数组成员作为自己的读写缓冲区。

>`NIO`的`ByteBuffer`局限性，如下，
- `ByteBuffer`长度固定，一旦完成分配，它的容量不能动态扩展和收缩，当需要编码的对象大于`ByteBuffer`的容量时，会发生索引越界异常；
- `ByteBuffer`只有一个标识位置的指针`position`，读写的时候需要手工调用`flip()`和`rewind()`等，使用者必须小心谨慎地处理这些`API`，否则很容易导致程序处理失败；
- `ByteBuffer`的`API`功能有限，一些高级和实用的特性它不支持，需要使用者自己编程实现。
{: .prompt-tip }

由于上面的局限，才有了`Netty`自己封装的`ByteBuf`。为了更高效地操作内存缓冲区，`Netty`提供了`ByteBuf`缓冲区来替代`NIO`的`ByteBuffer`。

## **ByteBuf是什么**

`ByteBuf`是一个字节容器，内部是一个字节数组，从逻辑上来分，字节容器内部分为四个部分，
- 第一部分是已用字节，表示已经使用完的废弃无效字节；
- 第二部分是可读字节，表示保存的有效数据，从`ByteBuf`读取的数据都来自这一部分；
- 第三部分是可写字节，写入到`ByteBuf`的数据都会写到这一部分；
- 第四部分是可扩容字节，表示`ByteBuf`最多还能扩容的大小。

![Desktop View](/assets/images/20260705/netty_bytebuf_fields.png){: width="500" height="250" }
_ByteBuf定义_

`ByteBuf`定义了三个整型属性，有效地区分可读数据和可写数据的索引，使得读写之间相互没有冲突。这三个属性定义在`AbstractByteBuf`抽象类中，分别是`readerIndex`（读指针）、`writerIndex`（写指针）、`maxCapacity`（最大容量）。

>与`NIO`的`ByteBuffer`相比，`ByteBuf`的优势如下，
- 读取和写入索引分开（`ByteBuf`对`NIO`的`ByteBuffer`进行了改良，增加了一个读写索引，引入读写下标`readerIndex`和`writerIndex`，分别用于读写的控制）
- 支持零复制（比如复合缓冲区类型），减少了内存复制
- `Pooling`（池化加速，减少了内存分配和释放，提升了效率）
- 不需要调用`flip()`方法去切换读/写模式
- 可扩展性好
- 可以自定义缓冲区类型
- 方法的链式调用
- 可以进行引用计数，方便重复使用
{: .prompt-tip }

## **ByteBuf的分类**

从所属的内存区域区分，`ByteBuf`和`NIO`的`ByteBuffer`类似，分为堆缓冲区和直接缓冲区，
- ‌堆缓冲区(`HeapByteBuf`)‌，将数据存储在`JVM`堆空间中，特点是在内存的分配和回收速度快，可以被`JVM`自动回收，缺点是如果进行`socket`的`IO`读写，需要额外一次内存复制，将堆内存对应的缓冲区复制到内核`Channel`中，性能会有一定程度的下降。
- 直接内存(`DirectByteBuf`)‌，非堆内存，在堆外进行内存分配，相比于堆内存，分配和回收速度会慢一些，进行`socket`的`IO`读写将它写入或从`socket channel`中读取时，由于少一次内存复制，速度比堆内存快。

从内存回收角度看，`ByteBuf`分为两类，
- 基于`pooled`的`ByteBuf`，使用内存池进行分配和回收管理，达到重复利用的效果；
- 普通的`ByteBuf`，借助`JVM`的垃圾回收机制进行内存管理。

两者的区别是基于`pooled`的`ByteBuf`可以重复利用，降低由于高负载导致的频繁`GC`。这里有两个池，对象池和内存池。

>`ByteBuf`的常用类型，`Heap`、`Direct`、`pooled`、`Unpooled`四个特性之间的组合，如下，
- `UnpooledHeapByteBuf`
- `UnpooledDirectByteBuf`
- `PooledHeapByteBuf`
- `PooledDirectByteBuf`
{: .prompt-tip }

## **ByteBuf的核心类**

核心顶层抽象类是`ByteBuf`，`AbstractByteBuf`继承了`ByteBuf`类，实现了它的大部分抽象方法，

![Desktop View](/assets/images/20260705/bytebuf_class_inheritance_relation.png){: width="800" height="400" }
_ByteBuf的继承关系_

### **模板模式**

>`AbstractByteBuf`模板模式<br>
`AbstractByteBuf`类中，采用模板方法的设计模式，真正内存读写的细节，以钩子的形式，延迟到子类实现。<br>
父子类的职责划分
- 父类是默认抽象实现类，负责`ByteBuf`操作的骨架；
- 子类负责骨架里面钩子方法的实现。
{: .prompt-tip }

`AbstractByteBuf`类负责下标检查和移动，其他公共的操作。比如`writerIndex`、`readerIndex`的操作。

不同的子类，内存结构不一样，`AbstractByteBuf`子类负责真正进行内存读写，比如`writeBytes`、`getBytes`。因为各种`ByteBuf`读写的内存区域不同，读写方式也就有差异。

举个例子，将指定字节数组写入到`ByteBuf`中，分两步，
- 把要写的字节`src`写入到内部的内存结构，`setBytes`是一个钩子方法，由子类实现；
- 调整写索引，写的位置由基类方法来完成。

```java
public ByteBuf writeBytes(byte[] src, int srcIndex, int length)
{
    ensureWritable(length);
    //抽象钩子方法，由不同的子类根据自身的内存类型（堆内、堆外等）实现差异化的读写逻辑
    setBytes(writerIndex, src, srcIndex, length);
    writerIndex += length;
    return this;
}
```

同样地，读取内容，将`ByteBuf`内容读取到指定字节数组，

```java
public ByteBuf readBytes(byte[] dst, int dstIndex, int length)
{
    checkReadableBytes(length);
    //抽象的钩子方法： 真正读内存的getBytes
    getBytes(readerIndex, dst, dstIndex, length);
    readerIndex += length;
    return this;
}
```

### **引用计数**

`AbstractReferenceCountedByteBuf`，`AbstractByteBuf`的直接子类，提供了引用计数的功能，所有的`Netty`子类都继承了引用计数基类，其所有的子类都可以使用该功能防止内存泄露。换句话说，`Netty`的`ByteBuf`的内存回收工作是通过引用计数的方式管理的。

`Netty`之所以采用计数器来追踪`ByteBuf`的生命周期，一是能对`Pooled ByteBuf`提供支持，二是能够尽快地发现那些可以回收的`ByteBuf`（非`Pooled`），以便提升`ByteBuf`的分配和销毁的效率。

>什么是`Pooled`（池化）的`ByteBuf`缓冲区？<br/>
从`Netty4`版本开始，新增了`ByteBuf`的池化机制。即创建一个缓冲区对象池，将没有被引用的`ByteBuf`对象，放入对象缓存池中；当需要时，则重新从对象缓存池中取出，而不需要重新创建。
{: .prompt-tip }

引用计数的两个核心方法，
- `retain()`将引用计数器加`1`；
- `release()`将引用计数器减去`1`。

在`Netty`中，如果引用为`0`，表示缓冲区不能再继续使用，再次访问这个`ByteBuf`对象，将会抛出异常；如果引用为`0`，表示这个`ByteBuf`没有哪个进程引用它，它占用的内存需要回收。

下面是一个使用引用计数的例子，

```java
ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();
Logger.info("after create:" + buffer.refCnt());

buffer.retain();
Logger.info("after retain:" + buffer.refCnt());

buffer.release();
Logger.info("after release:" + buffer.refCnt());

buffer.release();
Logger.info("after release:" + buffer.refCnt());

//错误:refCnt: 0,不能再retain
buffer.retain();
Logger.info("after retain:" + buffer.refCnt());
```

为了确保引用计数不会混乱，在`Netty`的业务处理器开发过程中，应该坚持一个原则，`retain`和`release`方法应该结对使用。对缓冲区调用了一次`retain`，就应该调用一次`release`。

>怎么保障引用计数的原子性？<br/>
引用计数保持在`refCnt`字段里边，使用字段的原子更新器来做增减。
{: .prompt-tip }

`AbstractReferenceCountedByteBuf`的原子操作
- `refCnt`：保存引用计数的字段
- `REFCNT_FIELD_OFFSET`：`refCnt`字段在内存中的地址偏移量
- `refCntUpdater`：`refCnt`字段的原子更新器

![Desktop View](/assets/images/20260705/AbstractReferenceCountedByteBuf_class_code.png){: width="800" height="400" }
_AbstractReferenceCountedByteBuf类定义_

![Desktop View](/assets/images/20260705/AbstractReferenceCountedByteBuf_class_retain.png){: width="800" height="400" }
_AbstractReferenceCountedByteBuf retain_

![Desktop View](/assets/images/20260705/AbstractReferenceCountedByteBuf_class_release.png){: width="800" height="400" }
_AbstractReferenceCountedByteBuf release_

>`AtomicIntegerFieldUpdater`是对`Object`的指定字段进行原子更新。<br/>
`AtomicIntegerFieldUpdater`是`JUC`包下的工具类，和`AtomicInteger`等原子变量在一个包下，底层使用的是`CAS`自旋锁机制，保障操作的原子性。
{: .prompt-tip }

### **ByteBuf的浅层复制**

`ByteBuf`的浅层复制分两种，`duplicate`（整体浅层复制）和`slice`（切片浅层复制），`duplicate`和`slice`方法都是浅层复制。不同的是，`slice`方法是切取一段的浅层复制，而`duplicate`是整体的浅层复制。

浅层复制方法不会实际去复制数据，也不会改变`ByteBuf`的引用计数，这就会导致一个问题，在源`ByteBuf`调用`release`之后，一旦引用计数为零，就变得不能访问了；在这种场景下，源`ByteBuf`的所有浅层复制实例也不能进行读写了；如果强行对浅层复制实例进行读写，则会报错。

因此，在调用浅层复制实例时，可以通过调用一次`retain`方法来增加引用，表示它们对应的底层内存多了一次引用，引用计数为`2`。在浅层复制实例用完后，需要调用两次`release`方法，这样就不影响源`ByteBuf`的内存释放。

派生缓冲区分为池化、非池化两种，有不同的派生关系，派生缓冲区会产生自己的读写索引和其他标记索引，但是与原生缓冲区共享内部的数据（比如引用计数不是独立的）。

![Desktop View](/assets/images/20260705/bytebuf_derived_relation.png){: width="700" height="350" }
_派生缓冲区_

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_derived_relation.png){: width="400" height="200" }
_UnpooledDirectByteBuf派生缓冲区_

## **ByteBuf的自动创建和自动释放**

`ByteBuf`的自动创建、自动释放过程，被`Netty`封装起来了。

### **自动创建**

首先找到`ByteBuf`自动创建的最源头（`Channel`入站数据的读取，`NioSocketChannel`），数据进入到通道后，数据就是通道数据，怎么从通道中读取出来，这个问题回顾下前面的`NioEventLoop`反应器核心原理。

在`processSelectedKeys`里边，第一个参数就是选择键实例，第二个参数是选择键的附件（事件所属的通道），`ch.unsafe()`方法，拿到`Netty`内部`Nio`通道，`NioUnsafe`专门用来做`Java`通道的读写，事件类型的判断，选择键是可写的，就对`Java`的`NIO`通道做刷新操作，选择键是可读的，就做`Java`的`NIO`通道的读取工作，

![Desktop View](/assets/images/20260627/NioEventLoop_processSelectedKey.png){: width="800" height="400" }
_NioEventLoop.processSelectedKey事件分发环节_

这里就是入站数据最源头，
- `config`方法获取通道配置示例；
- `pipeline`获取到通道处理器的流水线；
- `config.getAllocator`方法，使用通道配置实例，获取到内存分配器；
- 前面准备工作做好后，读取数据之前，直接分配`bytebuf`（存放数据的缓冲区），分配大小根据公式推断出合理的大小，进行通道数据读取，从`socket`里边接收数据。

![Desktop View](/assets/images/20260627/AbstractNioByteChannel_NioByteUnsafe_read.png){: width="600" height="300" }
_OP-READ事件对应的NioByteUnsafe.read方法实现_

上面的方法，通过通道获取到了很多属性，`Netty`对`JDK`原生的`ServerSocketChannel`进行了封装和增强，拥有了如下的属性，
- `id`，标识唯一身份信息
- 可能存在的`parent Channel`
- 管道，`pipeline`
- 用于数据读写的`unsafe`内部类，如（封装`JDK`的`JdkChannel`）
- 关联上相伴终生的`NioEventLoop`（反应器）
- `channelconfig`配置实例

内存分配器，可以在引导类上面进行配置，这里使用的是非池化的内存分配器，这里配置上之后，就可以在前面的事件处理方法中直接调用到，

![Desktop View](/assets/images/20260627/channel_config_case.png){: width="600" height="300" }
_ChannelConfig通道配置类示例_

`doReadBytes`方法，这个方法就是把`Java`通道的数据读取到`ByteBuf`里，这个`Java`通道就是被`Netty`通道封装起来的，完成入站数据的读取，

![Desktop View](/assets/images/20260627/NioSocketChannel_doReadBytes.png){: width="800" height="400" }
_读取NioSocketChannel_

`read`方法的后半部分，把读取到的`ByteBuf`，通过`Channel`的处理器流水线分发出去，进入流水线进行`Handler`处理，不是只读一次，会判断进行持续读取，流水线`Handler`处理，就是`Netty`的`Handler`开发，
- 触发`pipeline.fireChannelRead(byteBuf)`（通过`pipeline`触发读事件）。
- 判断是否继续读：有两个标准，
    - 一是不能超过最大的读取次数（默认`16`次）；
    - 二是缓冲区的数据每次都要读满，比如分配`2KB ByteBuf`，则必须读取`2KB`的数据。

![Desktop View](/assets/images/20260627/AbstractNioByteChannel_NioByteUnsafe_read.png){: width="600" height="300" }
_关注NioByteUnsafe.read方法实现的后半部分_

`ByteBuf`的后续观察中，用到了内存缓冲区的分配器。分配器在分配内存缓冲区时，会调用`ioBuffer`方法，会根据操作系统平台的环境进行判断，是不是有`unsafe`类，如果没有，就直接分配堆内存，如果有，就直接分配直接缓冲区。

![Desktop View](/assets/images/20260705/AbstractByteBufAllocator_ioBuffer.png){: width="500" height="250" }
_AbstractByteBufAllocator ioBuffer_

分配直接缓冲区，一类是池化，一类是非池化。默认情况下，进入到池化的分配器子类，分配池化的直接缓冲区，

![Desktop View](/assets/images/20260705/PooledByteBufAllocator_newDirectBuffer.png){: width="800" height="400" }
_PooledByteBufAllocator newDirectBuffer_

>总结下`NioByteUnsafe.read`方法的工作，
- 分配缓冲区：默认`1024 Byte`，之后根据最近几次请求的数据包大小，猜测下一次数据包大小；
- 读取数据：直接调用`Java NIO`的底层代码；
- 触发`pipeline.fireChannelRead(byteBuf)`：开始`pipeline`流水线的业务处理；
- 判断是否继续读：有两个标准，一是不能超过最大的读取次数（默认`16`次）；二是缓冲区的数据每次都要读满，比如分配`2KB ByteBuf`，则必须读取`2KB`的数据。
{: .prompt-tip }

### **自动释放**

第一种，**流水线尾部的自动释放**。

如果流水线前面的处理器没有对`ByteBuf`释放，尾部的最后一个环节，会做一个释放工作。当然，如果在流水线的某一个环节，把`ByteBuf`释放掉，尾部的也不会再释放。

![Desktop View](/assets/images/20260705/pipeline_tail_release.png){: width="600" height="300" }
_流水线尾部的自动释放（图片来自网络）_

怎么让`ByteBuf`到达尾部？
通过调用`super.channelRead`方法，一路传递到尾部。

第二种，**处理器不是继承于`ChannelInboundHandler`，而是继承于`SimpleChannelInboundHandler`（`Netty`自定义的基类）**。

`SimpleChannelInboundHandler`基类帮我们做了释放工作，继承了这个基类，业务代码是写在`channelRead0`方法里边的，不是写在`channelRead`里边。

第三种，**出站处理时，自动释放**。

之前都是从前向后，现在这种是从后往前，最后会走到`HeadContext`，但通道的写入最后会在`AbstractNioByteChannel`的`doWrite`方法，把`ByteBuf`内容写入到被封装的底层`Java NIO`通道，然后通过底层的`IOUtils`方法，把数据写入到直接内存，然后写入到内核缓冲区，直接发送出去。发送完之后，就会调用`in.remove`方法，这里就会释放`ByteBuf`。

![Desktop View](/assets/images/20260705/pipeline_head_release.png){: width="600" height="300" }
_出站处理时，自动释放（图片来自网络）_

![Desktop View](/assets/images/20260705/AbstractNioByteChannel_doWrite.png){: width="600" height="300" }
_AbstractNioByteChannel doWrite方法_

![Desktop View](/assets/images/20260705/AbstractNioByteChannel_doWriteInternal.png){: width="600" height="300" }
_AbstractNioByteChannel doWriteInternal方法_

![Desktop View](/assets/images/20260705/ChannelOutboundBuffer_remove.png){: width="600" height="300" }
_ChannelOutboundBuffer remove方法_

## **UnpooledHeapByteBuf非池化的堆内存**

非池化的堆内存，一个非池化的、内存空间分配在堆的`ByteBuf`，是`ByteBuf`最简单的一个实现类。

![Desktop View](/assets/images/20260705/UnpooledHeapByteBuf_class_inheritance_relation.png){: width="300" height="150" }
_ByteBuf的继承关系_

`UnpooledHeapByteBuf`的关键属性如下，

```java
// ByteBuf的分配器，ByteBufAllocator接口的所有实现类都是线程安全的
private final ByteBufAllocator alloc;

// 支撑数组，堆内字节缓冲，本质上是字节数组的封装
// 数据都存放在这个数组里
byte[] array;

// JDK Nio包中的ByteBuffer对象
// 主要用来做一些临时性的操作
private ByteBuffer tmpNioBuf;
```

### **UnpooledHeapByteBuf内存操作的钩子实现**

回到模板的抽象基类，`AbstractByteBuf`子类负责骨架里面钩子方法的实现，比如`setBytes`、`getBytes`方法。

`setBytes`使用`arraycopy`方法，通过数组拷贝的方式，直接将源字节流，拷贝到`UnpooledHeapByteBuf`对象的`array`属性中，

![Desktop View](/assets/images/20260705/UnpooledHeapByteBuf_setBytes.png){: width="600" height="300" }
_UnpooledHeapByteBuf setBytes方法_

`getBytes`则是将内部的字节数组`array`，拷贝到目标数组`dst`，

![Desktop View](/assets/images/20260705/UnpooledHeapByteBuf_getBytes.png){: width="600" height="300" }
_UnpooledHeapByteBuf getBytes方法_

### **UnpooledHeapByteBuf的内存分配**

`UnpooledByteBuf`的创建，不建议使用构造器创建，而是使用`Allocator`分配器创建。

`UnpooledByteBuf`默认的`Allocator`分配器，`UnpooledByteBufAllocator.DEFAULT`，

![Desktop View](/assets/images/20260705/UnpooledByteBufAllocator_craete_case.png){: width="800" height="400" }
_UnpooledByteBufAllocator创建用例_

运行结果如下，

```markdown
after ===========allocate ByteBuf(9, 100)============
capacity(): 9
maxCapacity(): 100
readerIndex(): 0
readableBytes(): 0
isReadable(): false
writerIndex(): 0
writableBytes(): 9
isWritable(): true
maxWritableBytes(): 100
```

具体做分配时，进一步调用钩子函数，最终通过`newHeapBuffer`来创建`UnpooledHeapByteBuf`实例，

![Desktop View](/assets/images/20260705/AbstractByteBufAllocator_heapBuffer.png){: width="600" height="300" }
_AbstractByteBufAllocator heapBuffer方法_

`newHeapBuffer`，由子类实现，

![Desktop View](/assets/images/20260705/UnpooledByteBufAllocator_newHeapBuffer.png){: width="800" height="400" }
_UnpooledByteBufAllocator newHeapBuffer方法_

`UnpooledByteBufAllocator`的内部类，根据`PlatformDependent.hasUnsafe`进行区分，创建不同的子类对象，
- 内部类`InstrumentedUnpooledHeapByteBuf`继承了`UnpooledHeapByteBuf`；
- 内部类`InstrumentedUnpooledUnsafeHeapByteBuf`继承了`UnpooledUnsafeHeapByteBuf`。

>`InstrumentedUnpooledUnsafeHeapByteBuf`和`InstrumentedUnpooledHeapByteBuf`是`UnpooledByteBufAllocator`的内部类。
{: .prompt-tip }

这里重点关注`UnpooledHeapByteBuf`、`UnpooledUnsafeHeapByteBuf`，

![Desktop View](/assets/images/20260705/UnpooledHeapByteBuf_inheritance_ralation.png){: width="500" height="250" }
_UnpooledHeapByteBuf子类的继承关系_

`UnpooledUnsafeHeapByteBuf`和`UnpooledHeapByteBuf`的不同之处，在于`allocateArray`分配字节数组的实现不同。
- `UnpooledUnsafeHeapByteBuf`
    - 优先使用性能更好的`jdk.internal.misc.Unsafe`类去分配和释放字节数组
- `InstrumentedUnpooledHeapByteBuf`
    - 使用`new byte[]`创建`byte`数组。

`UnpooledUnsafeHeapByteBuf`
- `UnpooledUnsafeHeapByteBuf`对字节数组的分配和释放，也通过`Unsafe`类来进行，
    - `jdk.internal.misc.Unsafe.allocateUninitializedArray(int)`
- `UnpooledUnsafeHeapByteBuf`后续对字节数组的读写，也通过`Unsafe`类来进行
    - 通过这种方式进行字节数组的读写，据说性能也会高些。
除此之外，`UnpooledUnsafeHeapByteBuf`和`UnpooledHeapByteBuf`没有其他差别。

`JDK 1.8`没有`allocateUninitializedArray`这个方法，这个方法有门槛，

通过`Unsafe.allocateUninitializedArray`来分配数组的条件，只有`JDK`版本为`9`以上并且持有`jdk.internal.misc.Unsafe`类，所以对于`Java 8`来说，还是通过`new byte[]`创建`byte`数组的方式分配内存。

对于`Java 9`及以上版本`HotSpot`虚拟机来说一般都会实例化这个类。如果不是`Java 9`以上版本，或者没有`jdk.internal.misc.Unsafe`，那么默认实例化`InstrumentedUnpooledHeapByteBuf`。

>这里的使用的`Unsafe`类不是`JDK1.8`中的`sun.misc.Unsafe`类，而是`jdk.internal.misc.Unsafe`类。
{: .prompt-tip }

`Netty`里边，对这些版本有做判断，`PlatformDependent.hasUnsafe`进行`Unsafe`存在性判定，

![Desktop View](/assets/images/20260705/PlatformDependent_hasUnsafe_version.png){: width="700" height="350" }
_版本有做判断_

`PlatformDependent0.ALLOCATE_ARRAY_METHOD`保持`Unsafe`的内存分配方法，`ALLOCATE_ARRAY_METHOD`常量，以反射式的模式初始化，初始化为以下方法，用于后续进行快速的堆内存分配。

```java
jdk.internal.misc.Unsafe.allocateUninitializedArray(int)
```

### **内存分配钩子allocateArray的实现**

接下来进入正题，介绍内存分配钩子`allocateArray`的实现。

基类`UnpooledHeapByteBuf`的内存分配钩子`allocateArray`如下，

```java
protected byte[] allocateArray(int initialCapacity) {
    return new byte[initialCapacity]; // 默认分配字节数组
}
```

两个子类，都重写了这个分配钩子，
- `InstrumentedUnpooledHeapByteBuf`
- `InstrumentedUnpooledUnsafeHeapByteBuf`

`UnpooledHeapByteBuf`的钩子实现就很简单，如上所示，`InstrumentedUnpooledHeapByteBuf`这个什么都没做，直接调用了`UnpooledHeapByteBuf`基类的数组分配方法，堆内存的内存数量增加了一个数组长度，做统计计数，

![Desktop View](/assets/images/20260705/InstrumentedUnpooledHeapByteBuf_allocateArray.png){: width="800" height="400" }
_InstrumentedUnpooledHeapByteBuf的内存分配钩子allocateArray_

`InstrumentedUnpooledUnsafeHeapByteBuf`这个则是完整的重写了，没有调用`UnpooledHeapByteBuf`基类的，

![Desktop View](/assets/images/20260705/InstrumentedUnpooledUnsafeHeapByteBuf_allocateArray.png){: width="600" height="300" }
_InstrumentedUnpooledUnsafeHeapByteBuf的内存分配钩子allocateArray_

根据一个门禁常量判断有没有`Unsafe`，去执行不同类型的方法分配内存，

![Desktop View](/assets/images/20260705/PlatformDependent_allocateUninitializedArray.png){: width="800" height="400" }
_PlatformDependent allocateUninitializedArray，根据门禁进行数组分配_

![Desktop View](/assets/images/20260705/UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD_constant.png){: width="800" height="400" }
_UNINITIALIZED ARRAY ALLOCATION THRESHOLD的门禁取值_

`UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD`门禁条件，
- `JDK`版本为`9`以上
- 并且持有`jdk.internal.misc.Unsafe`类的`allocateUninitializedArray`
- 否则，`UNINITIALIZED_ARRAY_ALLOCATION_THRESHOLD`的值为`-1`

`PlatformDependent0`通过`unsafe`分配字节数组

![Desktop View](/assets/images/20260705/PlatformDependent0_allocateUninitializedArray.png){: width="800" height="400" }
_PlatformDependent0通过unsafe分配字节数组_

总之，`UnpooledUnsafeHeapByteBuf`的数组分配和平台有关，
- 如果使用了`Java 9`以上版本，并且非安卓，`UnpooledUnsafeHeapByteBuf`的`allocateArray`通过`Unsafe`来分配初始的字节数组，
```java
jdk.internal.misc.Unsafe.allocateUninitializedArray(int)
```
- 其他情况下，直接创建`byte`数组并返回。

>`UnpooledHeapByteBuf`是一个基于`JVM`堆内存进行内存分配的缓冲区，是一个非池化的实现。在`IO`读写的时候多一次从`Linux`用户空间内存和`JVM`内存之间的复制过程，在每次`IO`读写的时候都会创建一个`UnpooledHeapByteBuf`对象。具有以下特点，
- 多了一次内存复制，`UnpooledHeapByteBuf`在`IO`读写的场景下性能较低；
- 如果需要频繁进行内存分配和回收，但相比于堆外内存申请和释放，成本要低一些。
{: .prompt-tip }

## **NIO的DirectByteBuffer**

在开始`UnpooledDirectByteBuf`之前，必须先介绍一下`NIO`的`DirectByteBuffer`。

看一个`UnpooledDirectByteBuf`的例子，调试查看`UnpooledDirectByteBuf`的内部，可以看到`Netty`的`UnpooledDirectByteBuf`封装了`NIO`的`DirectByteBuffer`，

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_test_case.png){: width="800" height="400" }
_UnpooledDirectByteBuf封装了NIO的DirectByteBuffer_

`JDK1.4`中开始，我们可以使用`native`方法在直接内存上来分配内存，并在`JVM`堆内存上维持一个引用来进行访问，当`JVM`堆内存上的引用被回收后，这块内存被操作系统回收。

具体来说，
- 直接内存并不是虚拟机运行时数据区的一部分，也不是`Java`虚拟机规范中定义的内存区域。
- 使用`native`函数库直接分配堆外内存，然后通过一个存储在`Java`堆里面的`DirectByteBuffer`对象作为这块内存的引用进行操作。
- 这样能在一些场景中显著提高性能，因为避免了在`Java`堆中和`native`堆中来回复制数据。

>直接内存与`JVM`堆内存的概念区别
- `JVM`堆内存‌
    - 堆内内存是由`JVM`所管控的`Java`进程内存，我们平时在`Java`中创建的对象都处于堆内内存中，并且它们遵循`JVM`的内存管理机制，`JVM`会采用垃圾回收机制统一管理它们的内存。
- 堆外内存（直接内存）是相对于堆内内存的一个概念，堆外内存就是存在于`JVM`管控之外的一块内存区域，因此它是不受`JVM`的管控。
{: .prompt-tip }

使用`NIO ByteBuffer`来分配直接内存，放`5`个数，转换模式后，读取出来，

![Desktop View](/assets/images/20260705/DirectByteBuf_test_case.png){: width="500" height="250" }
_NIO DirectByteBuffer的例子_

![Desktop View](/assets/images/20260705/DirectByteBuffer_inheritance_relation.png){: width="300" height="150" }
_NIO DirectByteBuffer的继承关系_

>`DirectByteBuffer`是`Java`用于实现堆外内存的一个重要类，我们可以通过该类实现堆外内存的创建、使用和销毁。
- `DirectByteBuffer`的创建
- `DirectByteBuffer`的性能优势
- 如何设置堆内存的大小
- `DirectByteBuffer`创建原理
- `DirectByteBuffer`释放原理
- `Unsafe`的内存分配和释放方法
- 堆外内存的使用场景
{: .prompt-tip }

### **DirectByteBuffer的创建方法**

| 类型 | 方法 |
| :--- | :--- |
| 堆缓冲（堆内存） | `ByteBuffer.allocate(int capacity)` |
| 直接缓冲区（直接内存） | `ByteBuffer.allocateDirect(int capacity)` |
| 堆缓冲（堆内存） | `ByteBuffer.wrap(byte[] array)` |
| 堆缓冲（堆内存） | `ByteBuffer.wrap(byte[] array, int offset, int length)` |

使用`ByteBuffer`静态类的`allocateDirect`静态方法，创建直接缓冲区`ByteBuffer`，其他三个都是堆内存，
- `allocate`方式创建的`ByteBuffer`对象，我们称之为非直接缓冲区，这个`ByteBuffer`对象（和对象包含的缓冲数组）都位于`JVM`的堆区。
- `allocateDirect`方法创建的`ByteBuffer`，我们称之为直接缓冲区，此时`ByteBuffer`对象本身在堆区，而缓冲数组位于非堆区，`ByteBuffer`对象内部存储了这个非堆缓冲数组的地址。
- `wrap`方式和数组创建方式创建的`ByteBuffer`没有本质区别，都创建的是非直接缓冲区。

### **DirectByteBuffer的性能优势**

`DirectByteBuffer`在非堆区的缓冲数组可以通过`JNI`（底层使用系统调用）方式进行`IO`操作，性能优势如下，
- `JNI`不受`GC`影响，机器码执行速度也比较快；
- 还避免了`JVM`堆区与进程用户空间缓冲区的数据拷贝，所以`IO`速度比非直接缓冲区快。

也有一些劣势，
- `allocateDirect`方式创建`ByteBuffer`对象花费的时间；
- 回收该对象花费的时间比较多，所以这个方法适用于创建那些需要重复使用的缓冲区对象。

>创建和回收该直接缓冲区对象性能低，为什么？<br/>
创建和回收内存的时候，是通过`JNI`系统调用来完成的，需要进行内核态和用户态之间的切换，比如分配内存用的`C`的`malloc`方法。<br/>
一般情况下，`allocateDirect`的`JNI`实现，涉及到`C`函数库中的内存分配函数`malloc()`，它具体是使用`sbrk()`系统调用来分配内存，当`malloc`调用`sbrk()`的时候就涉及一次从用户态到内核态的切换。
{: .prompt-tip }

`Java`程序里边，怎么配置堆外内存的大小？
默认情况下，堆外内存和`JVM`内存大小相同，

![Desktop View](/assets/images/20260705/DirectByteBuf_size_query.png){: width="800" height="400" }
_获取堆外内存的大小_

![Desktop View](/assets/images/20260705/DirectByteBuf_size_same_to_jvm_heap.png){: width="800" height="400" }
_默认与JVM堆大小相同，设置-Xmx100M_

>配置的最大堆内存是`100M`，实际拿到的最大堆内存是`96M`，为什么？<br/>
新生代占`1/3`，老生代占`2/3`，<br/>
新生代分`3`个区，一个是`Eden`，一个是`Survivor`，两个`Survivor`是来回复制的，所以只能算一个，`Eden`和`Survivor`比例`8:1:1`，`Eden`和`Survivor`占`9/10`。<br/>
`maxMemory = Eden + Survivor + Old Gen`（`96M = 30M + 66 M`）<br/>
{: .prompt-tip }

>如何设置堆内存的大小：`-XX:MaxDirectMemorySize=1024M`
{: .prompt-tip }

### **DirectByteBuffer创建原理**

`DirectByteBuffer`的构造函数主要做以下三个事情，
- 根据页对齐和`pageSize`来确定本次的要分配内存实际大小
- 实际分配内存，并且记录分配的内存大小
- 声明一个`Cleaner`对象用于清理该`DirectBuffer`内存

![Desktop View](/assets/images/20260705/DirectByteBuf_new.png){: width="800" height="400" }
_DirectByteBuf的构造函数_

传进来实际申请的数量，计算一个实际分配的数量，不是传进来申请多少，就分配多少，还得考虑内存对齐，尺寸大小的计算。

### **DirectByteBuffer的释放**

`DirectByteBuffer`内存回收主要有以下方式，
- 通过`FullGC`来自动回收
- 通过手动`System.gc`来回收
- 另一种是通过构造函数里创建的`Cleaner`对象来回收
- 通过`Unsafe.freeMemory`方法直接释放

看个`DirectByteBuf FullGC`的例子，虚拟机配置如下，

```markdown
// 直接内存设置最大1G
// 打印GC日志
-XX:MaxDirectMemorySize=1024M -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails
```

当`JVM`发现直接内存`MaxDirectMemorySize=1024M`所剩无几的时候，就会触发`FullGC`，

![Desktop View](/assets/images/20260705/DirectByteBuf_fullgc_case.png){: width="600" height="300" }
_DirectByteBuffer FullGC的例子_

现在把直接内存在`JVM`里设置在`1G`，每次申请`250M`内存，可以发现`3`个循环之后会触发一次`FullGC`，`GC`日志输出如下，

```markdown
VM.maxDirectMemory() 1024.0 MB
[Full GC (System.gc()) [CMS: 0K->1477K(87424K), 0.1183745 secs] 11007K->1477K(126720K), [Metaspace: 5065K->5065K(1056768K)], 0.1195972 secs] [Times: user=0.03 sys=0.09, real=0.12 secs] 
[Full GC (System.gc()) [CMS: 1477K->1137K(87424K), 0.0406728 secs] 1478K->1137K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0419954 secs] [Times: user=0.03 sys=0.02, real=0.04 secs] 
[Full GC (System.gc()) [CMS: 1137K->1134K(87424K), 0.0190550 secs] 1839K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0192406 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0113586 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0115312 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0191127 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0192708 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0124816 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0126392 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0157665 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0159360 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0116543 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0118291 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0108858 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0110087 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0123029 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0124361 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0078329 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0079431 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0082095 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0083218 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0112228 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0113256 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0076363 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0078986 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0086060 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0087987 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0093434 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0094636 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0120732 secs] 1134K->1134K(126848K), [Metaspace: 5068K->5068K(1056768K)], 0.0122867 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0099590 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0101044 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0142404 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0143844 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0097119 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0098441 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0102243 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0103650 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0109815 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0111438 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0091188 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0092453 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0100537 secs] 1836K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0102063 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0119528 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0120607 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0092390 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0093926 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0086915 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0088150 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0102697 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0104111 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0108851 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0110153 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0123193 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0124395 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0083704 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0084688 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1134K->1134K(87424K), 0.0081445 secs] 1134K->1134K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0083285 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
程序运行时间： 15003ms
Heap
 par new generation   total 39424K, used 701K [0x0000000081a00000, 0x00000000844c0000, 0x00000000ab390000)
  eden space 35072K,   2% used [0x0000000081a00000, 0x0000000081aaf728, 0x0000000083c40000)
  from space 4352K,   0% used [0x0000000083c40000, 0x0000000083c40000, 0x0000000084080000)
  to   space 4352K,   0% used [0x0000000084080000, 0x0000000084080000, 0x00000000844c0000)
 concurrent mark-sweep generation total 87424K, used 1134K [0x00000000ab390000, 0x00000000b08f0000, 0x0000000100000000)
 Metaspace       used 5088K, capacity 5242K, committed 5504K, reserved 1056768K
  class space    used 567K, capacity 594K, committed 640K, reserved 1048576K
```

![Desktop View](/assets/images/20260705/DirectByteBuf_system_gc_case.png){: width="800" height="400" }
_DirectByteBuffer通过手动System.gc()来回收_

`System.gc()`调用会显式地触发`FullGC`，同时对老年代和新生代、直接内存进行回收，

```markdown
VM.maxDirectMemory() 1024.0 MB
[Full GC (System.gc()) [CMS: 0K->1477K(87424K), 0.0388643 secs] 11007K->1477K(126720K), [Metaspace: 5056K->5056K(1056768K)], 0.0390786 secs] [Times: user=0.02 sys=0.03, real=0.04 secs] 
[Full GC (System.gc()) [CMS: 1477K->1322K(87424K), 0.0127459 secs] 1490K->1322K(126848K), [Metaspace: 5057K->5057K(1056768K)], 0.0128895 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1322K->1144K(87424K), 0.0167491 secs] 2037K->1144K(126848K), [Metaspace: 5058K->5058K(1056768K)], 0.0169601 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0119271 secs] 1859K->1144K(126848K), [Metaspace: 5059K->5059K(1056768K)], 0.0126592 secs] [Times: user=0.00 sys=0.02, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0116094 secs] 1157K->1144K(126848K), [Metaspace: 5061K->5061K(1056768K)], 0.0119911 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0102302 secs] 1157K->1144K(126848K), [Metaspace: 5062K->5062K(1056768K)], 0.0104904 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0119499 secs] 1157K->1144K(126848K), [Metaspace: 5069K->5069K(1056768K)], 0.0121445 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0105994 secs] 1157K->1144K(126848K), [Metaspace: 5070K->5070K(1056768K)], 0.0107187 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0113304 secs] 1157K->1144K(126848K), [Metaspace: 5070K->5070K(1056768K)], 0.0114684 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0104194 secs] 1157K->1144K(126848K), [Metaspace: 5070K->5070K(1056768K)], 0.0106316 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0128884 secs] 1851K->1144K(126848K), [Metaspace: 5070K->5070K(1056768K)], 0.0130314 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0151508 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0153258 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0175015 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0176691 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0157848 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0160059 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0119696 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0121271 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0132510 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0134609 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0157065 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0158796 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0150871 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0152505 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0173008 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0174803 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0160453 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0162010 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0130528 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0132114 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0115966 secs] 1846K->1144K(126848K), [Metaspace: 5071K->5071K(1056768K)], 0.0117359 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0125462 secs] 1846K->1144K(126848K), [Metaspace: 5072K->5072K(1056768K)], 0.0127110 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0107178 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0108750 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0121459 secs] 2547K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0122771 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0112280 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0114169 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0090373 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0092042 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0102973 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0104285 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0116177 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0117482 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0113139 secs] 1846K->1144K(126848K), [Metaspace: 5073K->5073K(1056768K)], 0.0114889 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0125177 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0126617 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0118334 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0120065 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0137146 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0138653 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0176295 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0178353 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0168078 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0169966 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0198929 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0200726 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0173743 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0176300 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0188363 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0190534 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0197586 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0199219 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0170999 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0172787 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0154172 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0155388 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0149408 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0150700 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0116016 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0117469 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0113373 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0115147 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0114547 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0115811 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0133796 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0135362 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0150022 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0151837 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0165591 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0167208 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0144162 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0145845 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1144K(87424K), 0.0148112 secs] 1846K->1144K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0149734 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1144K->1145K(87424K), 0.0185839 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0187494 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0182045 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0184219 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0177225 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0179134 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0178189 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0180543 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0140190 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0142418 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1147K(87424K), 0.0176175 secs] 1846K->1147K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0178323 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1147K->1146K(87424K), 0.0138031 secs] 1848K->1146K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0139799 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1146K->1145K(87424K), 0.0143173 secs] 1848K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0145060 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1146K(87424K), 0.0143045 secs] 1846K->1146K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0144551 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1146K->1147K(87424K), 0.0146641 secs] 1847K->1147K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0148633 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1147K->1148K(87424K), 0.0166404 secs] 1848K->1148K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0167807 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1148K->1145K(87424K), 0.0190820 secs] 1849K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0194078 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0171780 secs] 1847K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0174187 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0185020 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0186907 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0167993 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0169282 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0177797 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0179099 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0168266 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0169756 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0132231 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0133741 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0129281 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0130592 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0119076 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0120189 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1150K(87424K), 0.0109614 secs] 1846K->1150K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0111364 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1150K->1145K(87424K), 0.0133880 secs] 1851K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0135588 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0136729 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0138535 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0165481 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0167231 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0199082 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0201061 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1152K(87424K), 0.0175450 secs] 1846K->1152K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0179324 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1152K->1145K(87424K), 0.0180362 secs] 1854K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0182399 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1152K(87424K), 0.0173960 secs] 1846K->1152K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0175724 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1152K->1145K(87424K), 0.0183213 secs] 1854K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0184789 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1152K(87424K), 0.0157460 secs] 1846K->1152K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0159655 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1152K->1145K(87424K), 0.0154369 secs] 1854K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0156180 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0144819 secs] 1847K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0146510 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0128476 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0129976 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0146637 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0148493 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0146661 secs] 1846K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0148100 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1152K(87424K), 0.0141918 secs] 1847K->1152K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0143431 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1152K->1145K(87424K), 0.0154548 secs] 1854K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0157413 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0169995 secs] 1847K->1145K(126848K), [Metaspace: 5074K->5074K(1056768K)], 0.0172008 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0190515 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0192372 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0188673 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0191239 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0186910 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0188792 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0192899 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0195104 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0145434 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0147364 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0140586 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0141754 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0137579 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0139115 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0133921 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0135673 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0111670 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0112989 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0149480 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0151831 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0144447 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0146719 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [CMS: 1145K->1145K(87424K), 0.0139020 secs] 1847K->1145K(126848K), [Metaspace: 5075K->5075K(1056768K)], 0.0140795 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
程序运行时间： 1616ms
Heap
 par new generation   total 39424K, used 701K [0x0000000081a00000, 0x00000000844c0000, 0x00000000ab390000)
  eden space 35072K,   2% used [0x0000000081a00000, 0x0000000081aaf5f8, 0x0000000083c40000)
  from space 4352K,   0% used [0x0000000083c40000, 0x0000000083c40000, 0x0000000084080000)
  to   space 4352K,   0% used [0x0000000084080000, 0x0000000084080000, 0x00000000844c0000)
 concurrent mark-sweep generation total 87424K, used 1145K [0x00000000ab390000, 0x00000000b08f0000, 0x0000000100000000)
 Metaspace       used 5094K, capacity 5242K, committed 5504K, reserved 1056768K
  class space    used 568K, capacity 594K, committed 640K, reserved 1048576K
```

>`System.gc()`和`FGC`的问题
- `‌System.gc()`不靠谱‌
    - 大部分调优指南都设置了参数`-XX:+DisableExplicitGC`来禁用`System.gc()`
    - `System.gc()`命令大部分情况被禁用。`System.gc()`会显式直接触发`Full GC`，同时对老年代和新生代、直接内存进行回收。而一般情况下我们认为，垃圾回收应该是自动进行的，无需手工触发。如果过于频繁地触发垃圾回收对系统性能是没有好处的。因此虚拟机提供了一个参数`DisableExplicitGC`来控制是否手工触发`GC`。
- ‌依赖老年代`FGC`也是不靠谱的‌
    - 只有堆内的对象（`DirectByteBuffer`）回收之后，才会顺带回收直接内存，反过来，仅触发直接内存回收不会回收堆内对象。解决方案：手动释放。
- `JDK`在`DirectByteBuffer`中提供了`Cleaner`用来主动释放内存。
- 同时还有`Unsafe`的`freeMemory`方法也可以实现手动释放。
{: .prompt-tip }

通过反射直接拿到直接内存`Cleaner`实例，调用`clean`方法，

![Desktop View](/assets/images/20260705/DirectByteBuf_cleaner_gc_case.png){: width="600" height="300" }
_使用Cleaner手动来回收内存_

`DirectByteBuffer`通过`FullGC`来回收内存的，`DirectByteBuffer`会自己检测情况而调用`system.gc()`，但是如果参数中使用了`DisableExplicitGC`那么就无法回收该块内存了，`-XX:+DisableExplicitGC`标志自动将`System.gc()`调用转换成一个空操作，就是应用中调用`System.gc()`会变成一个空操作。那么，如果设置了就需要我们手动来回收内存了。

`Cleaner`对象回收的原理

在每次新建一个`DirectBuffer`对象的时候，会同时创建一个`Cleaner`对象，同一个进程创建的所有`DirectBuffer`对象跟`Cleaner`对象的个数是一样的，并且所有的`Cleaner`对象会组成一个链表，前后相连。

```java
public static Cleaner create(Object ob, Runnable thunk) {
    if (thunk == null)
        return null;
    return add(new Cleaner(ob, thunk));
}
```

`Cleaner`对象的`clean`方法执行时机是，`JVM`在判断该`Cleaner`对象关联的`DirectBuffer`已经不被任何对象引用了（也就是经过可达性分析判定为不可达的时候）。此时`Cleaner`对象会被`JVM`挂到`PendingList`上。然后有一个固定的线程扫描这个`List`，如果遇到`Cleaner`对象，那么就执行`clean`方法。

`Cleaner`对象回收的问题：存在滞后性。默认情况下，`Netty`不是用`Cleaner`来释放内存的。

通过`Unsafe.freeMemory`方法直接释放

如果通过`Unsafe.freeMemory`方法直接释放，那么需要使用`DirectByteBuffer`如下的构造器，

![Desktop View](/assets/images/20260705/DirectByteBuf_constructor.png){: width="600" height="300" }
_DirectByteBuf的构造函数_

前提是需要`reallocateMemory(long address, long size);`提前分配好内存。

这种方式不再需要`Cleaner`，构造器变了，`addr`参数是直接内存在用户空间里的内存地址。

### **sun.misc.Unsafe类的方法**

>`sun.misc.Unsafe`类<br/>
`Unsafe`类是在`sun.misc`包下，不属于`Java`标准。但是很多`Java`的基础类库，包括一些被广泛使用的高性能开发库都是基于`Unsafe`类开发的，比如`Netty`、`Cassandra`、`Hadoop`、`Kafka`等。<br/>
`Unsafe`类在提升`Java`运行效率，增强`Java`语言底层操作能力方面起了很大的作用。
{: .prompt-tip }

`sun.misc.Unsafe`提供了一组`JNI`方法来进行堆外内存的分配，重新分配，以及释放。这些方法里边分配内存，释放内存，都是用户空间里边的内存地址，不是`JVM`堆里边的内存，

```java
// 分配一块内存空间
public native long allocateMemory(long size);

// 重新分配一块内存，把数据从 address 指向的缓存中拷贝到新的内存块
public native long reallocateMemory(long address, long size);

// 释放内存
public native void freeMemory(long address);

// 以上方法也不是静态的，不能通过Unsafe.allocateMemory调用
```

获取`Unsafe`实例的方法，在使用`Unsafe`类的时候，

```java
Unsafe f = Unsafe.getUnsafe();
```

发现还是被拒绝了，抛出异常，

```markdown
java.lang.SecurityException: Unsafe
```

只能使用反射来做这件事，

```java
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe unsafe = (Unsafe) theUnsafe.get(null);
```

![Desktop View](/assets/images/20260705/unsafe_test_case.png){: width="600" height="300" }
_Unsafe的例子_

释放这部分内存的话，需要调用`freeMemory`或者`reallocateMemory`方法。

`DirectByteBuffer`使用`sun.misc.Unsafe`类申请直接内存，

![Desktop View](/assets/images/20260705/DirectByteBuf_new.png){: width="800" height="400" }
_DirectByteBuf的构造函数_

`DirectByteBuffer`使用`sun.misc.Unsafe`类释放直接内存，

![Desktop View](/assets/images/20260705/DirectByteBuf_free.png){: width="600" height="300" }
_DirectByteBuf的释放内存_

### **堆外内存的使用场景**

使用堆外内存的原因
- 对垃圾回收停顿的改善：因为`FullGC`意味着彻底回收，彻底回收时，垃圾收集器会对所有分配的堆内内存进行完整的扫描，这意味着一个重要的事实，这样一次垃圾收集对`Java`应用造成的影响，跟堆的大小是成正比的。过大的堆会影响`Java`应用的性能。如果使用堆外内存的话，堆外内存是直接受操作系统管理（而不是虚拟机）。这样做的结果就是能保持一个较小的堆内内存，以减少垃圾收集对应用的影响。
- 在某些场景下可以提升程序`IO`操纵的性能，省去了将数据从堆内内存拷贝到堆外内存的步骤。

什么情况下使用堆外内存？
- 堆外内存适用于生命周期中等或较长的对象（如果是生命周期较短的对象，在`YGC`的时候就被回收了，就不存在大内存且生命周期较长的对象在`FGC`对应用造成的性能影响）。
- 直接的文件拷贝操作，或者`IO`操作。直接使用堆外内存就能省去内存从用户内存拷贝到系统内存的操作，因为`IO`操作是系统内核内存和设备间的通信，而不是通过程序直接和外设通信的。
- 同时，还可以使用**池 + 堆外内存**的组合方式，来对生命周期较短，但涉及到`IO`操作的对象进行堆外内存的再使用。（`Netty`中就使用了该方式）

堆外内存，创建和释放比较耗时间，需要做到尽可能地复用，所以需要使用内存池来做内存管理。在`Netty`中，除了使用内存池，还使用了对象池，对象池主要用于两类对象，
- 生命周期较短，且结构简单的对象，在内存池中重复利用这些对象能增加`CPU`缓存的命中率，从而提高性能；
- 加载含有大量重复对象的大片数据，此时使用内存池能减少垃圾回收的时间。

堆外内存和内存池一样，也能缩短垃圾回收时间，但是它适用的对象和内存池完全相反。内存池往往适用于生命期较短的可变对象，而生命期中等或较长的对象，正是堆外内存要解决的。

堆外内存的特点
- 对于大内存有良好的伸缩性
- 对垃圾回收停顿的改善可以明显感觉到
- 在进程间可以共享，减少虚拟机间的复制

堆外内存的一些问题
- 堆外内存回收问题，不受`JVM`内存回收管理，分配回收成本较高
- 以及堆外内存的泄漏问题
    - 因此它的大小不会直接受限于`-Xmx`指定的最大堆大小，但是系统内存是有限的，`Java`堆和直接内存的总和依然受限于操作系统能给出的最大内存。
- 堆外内存的数据结构问题

堆外内存最大的问题，就是你的数据结构变得不那么直观，如果数据结构比较复杂，就要对它进行串行化（`serialization`），而串行化本身也会影响性能。另一个问题是由于你可以使用更大的内存，你可能开始担心虚拟内存（即硬盘）的速度对你的影响了。

## **UnpooledDirectByteBuf非池化的直接内存**

`Netty`非池化的直接内存

>直接内存`DirectByteBuffer`<br/>
`DirectByteBuffer`，顾名思义是直接内存（`Direct Memory`）上的`Byte`缓存区，直接内存不是`JVM Runtime`数据区域的一部分，也不是`Java`虚拟机规范中定义的内存区域。简单的说，这部分就是机器内存，分配的大小等都和虚拟机限制无关。
{: .prompt-tip }

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_test_case_02.png){: width="800" height="3400" }
_UnpooledDirectByteBuf noUnsafe的例子_

`AbstractByteBufAllocator`的`directBuffer`方法，用来分配直接内存类型的`ByteBuf`缓冲区，是`Netty`实现堆外内存操作的关键逻辑入口。

![Desktop View](/assets/images/20260705/AbstractByteBufAllocator_directBuffer.png){: width="800" height="400" }
_DirectByteBuf的directBuffer方法_

![Desktop View](/assets/images/20260705/UnpooledByteBufAllocator_newDirectBuffer.png){: width="800" height="400" }
_UnpooledByteBufAllocator newDirectBuffer_

两种平台场景
- `PlatformDependent.hasUnsafe()`
    - `hotspot JVM`，普通的`JVM`
- `!PlatformDependent.hasUnsafe()`
    - 安卓的`JVM`，`System.setProperty("java.vm.name","Dalvik");`
    - 或者配置环境变量，`System.setProperty("io.netty.noUnsafe","true");`

`PlatformDependent`与`PlatformDependent0`
- 主要针对操作系统、`JDK`版本等环境因素进行判断，判断是否支持堆外内存`Unsafe`以及一些关联类；
- 通过封装`Unsafe`申请堆外内存、释放、获取数据等操作。

### **!PlatformDependent.hasUnsafe()场景**

#### **UnpooledDirectByteBuf的内存分配**

在`!PlatformDependent.hasUnsafe()`这种场景，`UnpooledDirectByteBuf`构造函数会调用分配内存的钩子方法`allocateDirect`方法，

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_construct.png){: width="700" height="350" }
_UnpooledDirectByteBuf构造函数_

![Desktop View](/assets/images/20260705/InstrumentedUnpooledDirectByteBuf_allocateDirect.png){: width="700" height="350" }
_子类的allocateDirect钩子实现_

`UnpooledDirectByteBuf`的默认`allocateDirect`内存分配，在`allocateDirect`方法中，分配`NIO`直接内存。

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_allocateDirect.png){: width="600" height="300" }
_UnpooledDirectByteBuf的默认allocateDirect内存分配_

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_derived_relation.png){: width="400" height="200" }
_UnpooledDirectByteBuf的继承关系_

`UnpooledDirectByteBuf`的关键属性如下，

```java
//内部字节Java NIO buf，对java.nio.DirectByteBuffer的封装
private ByteBuffer buffer;

//临时nio buf
private ByteBuffer tmpNioBuf;

//容量
private int capacity;

//是否需要释放内存
private boolean doNotFree;
```

封装了`NIO`的`DirectByteBuffer`，调用其构造器进行创建时，就会创建`java.nio.DirectByteBuffer`，从而分配到位于堆外的直接内存。

分配内存的时候，委托给`NIO`的`ByteBuffer`分配直接内存（`allocateDirect`方法），

```java
protected ByteBuffer allocateDirect(int initialCapacity) {
    return ByteBuffer.allocateDirect(initialCapacity);
}
```

`java.nio.ByteBuffer`的`allocateDirect`方法，如下，

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

`NIO`的`ByteBuffer`的知识很重要，在`Android`平台，或者没有`Unsafe`类的情况下，`Netty`的`DirectByteBuf`封装了`NIO`的`ByteBuffer API`，这个场景比较简单。

#### **UnpooledDirectByteBuf内存操作的钩子实现**

回到模板的抽象基类，`AbstractByteBuf`子类负责骨架里面钩子方法的实现，比如`setBytes`、`getBytes`方法。

`AbstractByteBuf writeBytes`将指定字节数组写入到本`ByteBuf`中，`UnpooledDirectByteBuf`的`setBytes`钩子实现

```java
public ByteBuf setBytes(int index, byte[] src, int srcIndex, int length) {
    checkSrcIndex(index, length, srcIndex, src.length);

    //UnpooledDirectByteBuf的读写都转换为ByteBuffer的读写
    ByteBuffer tmpBuf = internalNioBuffer();

    tmpBuf.clear().position(index).limit(index + length);
    tmpBuf.put(src, srcIndex, length);
    return this;
}
```

代码中的`internalNioBuffer()`方法创建出一个`tmpBuf`，这个`tmpBuf`和源`ByteBuffer`共享原来的内存段，但是有独立的下标。

>`internalNioBuffer()`<br/>
`tmpBuf`是源`ByteBuffer`的浅层复制，对`tmpBuf`进行写入，变更会反映到源`buffer`属性。
{: .prompt-tip }

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_internalNioBuffer.png){: width="800" height="400" }
_UnpooledDirectByteBuf internalNioBuffer方法_

>`UnpooledDirectByteBuf`的`writerIndex`有谁负责修改？<br/>
`writerIndex`的修改在父类`writeBytes`模板方法中完成。
{: .prompt-tip }

`UnpooledDirectByteBuf`的`getBytes`钩子实现，`getBytes`也是转换为对`NIO ByteBuffer`的读取，

```java
private void getBytes(int index, ByteBuffer dst, boolean internal) {
    checkIndex(index, dst.remaining());

    ByteBuffer tmpBuf;
    if (internal) {
        tmpBuf = internalNioBuffer();
    } else {
        tmpBuf = buffer.duplicate();
    }
    // 使用clear方法，从写模式转成读取模式
    tmpBuf.clear().position(index).limit(index + dst.remaining());
    dst.put(tmpBuf);
}
```

### **PlatformDependent.hasUnsafe()场景**

在`PlatformDependent.hasUnsafe()`这种场景，分配内存会区分有无`Cleaner`，

![Desktop View](/assets/images/20260705/UnpooledDirectByteBuf_test_case_03.png){: width="800" height="400" }
_UnpooledDirectByteBuf Unsafe的例子_

![Desktop View](/assets/images/20260705/UnpooledByteBufAllocator_newDirectBuffer.png){: width="800" height="400" }
_UnpooledByteBufAllocator newDirectBuffer的钩子实现_

- `noCleaner`为`true`：创建`InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf`对象，简称`noCleaner`；
- `noCleaner`为`false`：创建`InstrumentedUnpooledUnsafeDirectByteBuf`对象，简称`hasCleaner`。

什么是`noCleaner`细分场景？
- `hasunsafe`
- 能够调用`unSafe.freeMemory(address)`去释放内存
- 能够反射调用`private DirectByteBuffer(long addr, int cap)`去分配内存

![Desktop View](/assets/images/20260705/UnpooledByteBufAllocator_construct.png){: width="800" height="400" }
_UnpooledByteBufAllocator的构造函数_

什么是`hasCleaner`细分场景？
- `hasunsafe`
- 不能反射调用`private DirectByteBuffer(long addr, int cap)`进行对象创建
- 能够反射调用`DirectByteBuffer(int cap)`去分配内存，创建对象
- 能够使用`DirectByteBuffer`的`Cleaner`的`clean`方法回收内存

>`noCleaner`与`hasCleaner`细分场景不同
- 构造器方式不同
    - `noCleaner`：反射调用`private DirectByteBuffer(long addr, int cap)`，双参数，类私有，无`cleaner`
    - `hasCleaner`：`new`操作调用`DirectByteBuffer(int cap)`，单参数，包私有，有`cleaner`
- 两个释放内存方式不同：
    - `noCleaner`：使用`unSafe.freeMemory(address)`完成内存释放
    - `hasCleaner`：使用`DirectByteBuffer`的`Cleaner`的`clean`方法回收内存
{: .prompt-tip }

>什么是`Cleaner`？
- `Cleaner`用来主动释放内存
    - `JDK`在`DirectByteBuffer`中提供了`Cleaner`用来主动释放内存，作用和`Unsafe`的`freeMemory`方法类似。
- `Cleaner`对象的构造时机
    - 在单参数、包私有的`DirectByteBuffer(int cap)`构造器中完成创建。
- `Cleaner`是个虚引用
    - 这个`Cleaner`属于虚引用类型，`DirectByteBuffer`创建它的时候，会将自身传入虚引用的构造函数中。当这个`DirectByteBuffer`不再被引用、即将被回收时，`GC`会把对应的`Cleaner`对象挂载到`PendingList`上。`JVM`中专门有一条`ReferenceHandler`线程会死循环扫描这个`List`，执行`Reference`的`tryHandlePending`方法，该方法会调用`pending`状态下`Cleaner`的`clean`方法，最终完成对应的直接内存回收操作。
{: .prompt-tip }

`hasCleaner`的`DirectByteBuffer(int cap)`，单参数，包私有，有`cleaner`，

![Desktop View](/assets/images/20260705/DirectByteBuf_new.png){: width="800" height="400" }
_DirectByteBuf的构造函数_

>`Cleaner`的性能问题<br/>
在运行过程中，`Cleaner`回收的内存，有一定的时间滞后。
`noCleaner`无论是在申请内存，还是释放内存，都比使用`hasCleaner`性能要好一点。
所以，`Netty`在`4.1`引入了`noCleaner`策略。
{: .prompt-tip }

`noCleaner`的`DirectByteBuffer`构造器，双参数，类私有，无`cleaner`的`DirectByteBuffer`构造器，

![Desktop View](/assets/images/20260705/DirectByteBuf_constructor.png){: width="600" height="300" }
_DirectByteBuf的构造函数_

`noCleaner`的双参数构造方法`DirectByteBuffer(long addr, int cap)`没调用`cleaner`的`clean`方法的。

![Desktop View](/assets/images/20260705/UnpooledUnsafeDirectByteBuf_inheritance_relation.png){: width="600" height="300" }
_UnpooledUnsafeDirectByteBuf的继承关系_

`NoCleaner`内存分配

```markdown
InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf的内存分配
--> InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf 构造器
--> UnpooledUnsafeNoCleanerDirectByteBuf 构造器
--> UnpooledUnsafeDirectByteBuf 构造器
--> InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf.allocateDirect
--> UnpooledUnsafeNoCleanerDirectByteBuf.allocateDirect
--> PlatformDependent.allocateDirectNoCleaner
```

`UnpooledUnsafeNoCleanerDirectByteBuf`内存分配总结
- 该类继承自`UnpooledUnsafeDirectByteBuf`
- 先通过`Unsafe`分配内存，再构造`DirectByteBuffer`对象
    - 分配直接内存时，是通过`Unsafe.allocateMemory(int capacity)`分配到直接内存，拿到直接内存起始地址`address`，再以`address`为参数，反射调用`DirectByteBuffer`的以下构造器创建`DirectByteBuffer`对象，
    - `DirectByteBuffer(long addr, int cap, Object ob)`
- 没有`cleaner`对象，调用`Unsafe`去释放

注意，`DirectByteBuffer`的这个构造器，不会创建`cleaner`对象，这也就是该类名有`NoCleaner`关键字的原因。既然没有`cleaner`，释放内存时也就不能通过`cleaner`来释放，所以释放内存时，是直接调用`Unsafe.freeMemory(long address)`来完成释放的。

看完内存分配，再看下`NoCleaner`的`setbytes`钩子实现。

`UnpooledUnsafeNoCleanerDirectByteBuf`的`setbytes`钩子实现，继承自`UnpooledUnsafeDirectByteBuf`，通过`unsafe`直接对内存进行读写，所以性能上会好一些。

![Desktop View](/assets/images/20260705/UnpooledUnsafeDirectByteBuf_setBytes.png){: width="600" height="300" }
_UnpooledUnsafeDirectByteBuf setBytes_

`hasCleaner`内存分配

```markdown
InstrumentedUnpooledUnsafeDirectByteBuf的内存分配
--> InstrumentedUnpooledUnsafeDirectByteBuf 构造器
--> UnpooledUnsafeDirectByteBuf 构造器
--> UnpooledUnsafeDirectByteBuf.allocateDirect
--> ByteBuffer.allocateDirect
```

`hasCleaner`的`setbytes`钩子实现

`InstrumentedUnpooledUnsafeDirectByteBuf`的`setbytes`钩子实现，继承自`UnpooledUnsafeDirectByteBuf`，通过`unsafe`直接对内存进行读写，所以性能上会好一些。

![Desktop View](/assets/images/20260705/UnpooledUnsafeDirectByteBuf_setBytes.png){: width="600" height="300" }
_UnpooledUnsafeDirectByteBuf setBytes_

>直接内存与堆内存简单对比
- 堆缓冲区可以不用池化
    - 堆缓冲区直接将数据存储在`JVM`堆空间中，它能在没有使用池化的情况下快速的分配和释放。
- 直接内存不池化会有性能损耗
    - 直接缓冲区是另外一种`ByteBuf`模式，它将缓冲区分配在堆外内存（非`JVM`运行时数据区），垃圾收集器不会管理这部分内存。
- 直接内存的优势
    - 它的优势在于网络数据传输场景下减少了一次内存复制，其劣势在于分配和释放都较为昂贵。
{: .prompt-tip }

所以，通常在`IO`通信线程的读写缓冲区使用`DirectByteBuf`（不过需要池化），后端业务消息的编解码模块使用`HeapByteBuf`（可以是非池化），可以达到很好的性能效果。

非池化看完了，接下来就是池化的内存。实际上在`Netty`里更多使用的是池化的内存。

## **PooledDirectByteBuf池化的直接内存**

从`Netty 4`开始，`Netty`加入了内存池管理，采用内存池管理比普通的`new ByteBuf`性能提高了数十倍。

![Desktop View](/assets/images/20260705/PooledByteBuf_inheritance_relation.png){: width="600" height="300" }
_PooledByteBuf的继承关系_

`PooledByteBuf`的分配
- 形的部分（`JVM`中`PooledByteBuf`对象）的分配
- 神的部分（实际的字节数组内存空间）的分配

形，说的是`PooledByteBuf`对象记录着池化内存的地址，偏移量这些属性；
神，保存着缓冲区的字节数组，对应直接内存，这一部分就是对应数据所在。

### **形的分配**

通过对象池，获取`PooledHeapByteBuf`对象，之后做重复利用，充分说明`Netty`的性能还是比较高的。

#### **PooledHeapByteBuf**

由于使用了对象池，每次创建`PooledHeapByteBuf`的时候，不是直接`new`，而是从对象池中去获取，然后设置引用计数器、读写`index`和缓冲区最大容量返回。

```java
static PooledHeapByteBuf newInstance(int maxCapacity) {
    PooledHeapByteBuf buf = RECYCLER.get();
    buf.reuse(maxCapacity);
    return buf;
}

final void reuse(int maxCapacity) {
    maxCapacity(maxCapacity);
    setRefCnt(1);
    setIndex0(0, 0);
    discardMarks();
}
```

池化的直接内存，套路也是一样的。

#### **PooledDirectByteBuf**

`PooledDirectByteBuf`使用了对象池，用`RECYCLER`来做回收重复利用。调用`memory`的`get`方法获取数据，`memory`是一个`DirectByteBuffer`对象。

```java
private static final Recycler<PooledDirectByteBuf> RECYCLER = new Recycler<PooledDirectByteBuf>() {
    @Override
    protected PooledDirectByteBuf newObject(Handle<PooledDirectByteBuf> handle) {
        return new PooledDirectByteBuf(handle, 0);
    }
};

static PooledDirectByteBuf newInstance(int maxCapacity) {
    PooledDirectByteBuf buf = RECYCLER.get();
    buf.reuse(maxCapacity);
    return buf;
}
```

### **神的分配（数据内存空间的分配）**

内存池的最顶层，可以理解为区，一个大的区，可以细分成很多`Chunk`块，一个块如果还太大的话，就细分为页。假设`1`个块`16M`，一个页`512K`等。

>`Pool`内存管理组件<br/>
在`Netty4`之后加入内存池管理，通过内存池管理比之前`ByteBuf`的创建性能得到了极大提高。
- ‌`PoolArena`竞技区‌
    - 在内存分配中，为了能够集中管理内存的分配及释放，同时提升分配和释放内存的性能，一般都会预先分配一大块连续的内存，不需要重复频繁地进行内存操作，这一大块连续的内存就叫做`memory Arena`，而`PoolArena`是`Netty`的内存池实现类。
- ‌`Chunk`块‌
    - 在`Netty`中，`PoolArena`是由多个`Chunk`组成的，而每个`Chunk`则由多个`Page`组成，`PoolArena`由`Chunk`和`Page`共同完成组织和管理工作。
- `PoolSubpage`页‌
    - 当进行小于一个`Page`的内存分配时，每个`Page`会被划分为大小相等的内存块，它的大小是根据第一次申请内存分配的内存块大小来决定的。一个`Page`只能分配与第一次申请的内存块大小相等的内存块，如果想要申请大小不相等的内存块，只能在新的`Page`上申请内存分配。`Page`中的存储区域的使用情况是通过一个`long`类型的数组`bitmap`来维护的，每一位表示一个区域的占用情况。
{: .prompt-tip }

#### **PoolThreadCache**

![Desktop View](/assets/images/20260705/PoolThreadCache.png){: width="300" height="150" }
_PoolThreadCache_

在每个线程去申请内存的时候，首先会通过`ThreadLocal`这种方式去获得当前线程的`PoolThreadCache`对象类型的`cache`对象，
`PoolThreadCache`一共分为两部分，一个是`cache`，在申请内存的时候首先会尝试从对象池中获取。

#### **ChunkList**

![Desktop View](/assets/images/20260705/ChunkList.png){: width="400" height="200" }
_ChunkList_

一个`arena`维护着多个`ChunkList`，每个`ChunkList`代表着它们内部的`Chunk`的使用率情况，这些`Chunk`会进行动态移动，每次完成内存分配后都会重新计算它们的使用率，判定其应当归属的`ChunkList`，随后将其移动到对应的列表中。每次执行内存分配操作时，会先通过指定算法定位到对应的`ChunkList`，再从该列表里挑选适配的`Chunk`完成内存分配。

#### **Chunk**

一个`Chunk`是`16M`，进行一次内存分配，不可能一次将一个`Chunk`全部分配，于是又将`Chunk`分割成更小的`Page`。

![Desktop View](/assets/images/20260705/chunk.png){: width="400" height="200" }
_Chunk_

一个`Chunk`会以`8K`的大小进行划分，分成一个个的`Page`，到时候分配内存的时候只需要以`Page`为单位进行内存划分。但是，如果我只需要一个`2K`大小的内存，那么分配给我一个`Page`岂不是又造成了浪费，于是又继续将`Page`进行划分成更小的`subpage`。

### **形和神的关系：内存池字节缓冲的表示**

它们都是`PoolChunk`分配到的一大段连续内存上的一小段，通过下标来控制访问范围。它们的主要属性在父类`PooledByteBuf`中，

```java
// memory，记录的是块的地址
protected T memory;
// offset，记录的是ByteBuf在块上的偏移量
protected int offset;
protected int length;
```

假设从同一个`chunk`上分配了两个堆字节缓冲，偏移量分别是`(offset=16 bytes,length=16 bytes)`和`(offset=32 bytes,length=16 bytes)`，那么他们是一大段连续内存上的两个片段，如图所示，

![Desktop View](/assets/images/20260705/chunk_case.png){: width="600" height="300" }
_Chunk示例_

### **PooledByteBufAllocator**

顾名思义，`PooledByteBufAllocator`就是一个用于分配`PooledByteBuf`的分配器。

线程的局部变量缓存，每个线程都有这样一个局部变量，`ThreadLocal`类型，

`PooledByteBufAllocator`主要属性有三个，

```java
private final PoolArena<byte[]> heapArenas;
private final PoolArena<ByteBuffer> directArenas;
private final PoolThreadLocalCache threadCache;
```

- `heapArenas`是堆内存的表示，每个`arena`内存以字节数组`byte[]`表示，注意`arena`不会直接持有内存，`arena`所有内存通过`PoolChunk`持有。
- `directArenas`是直接内存的表示，每个`arena`的内存以`DirectByteBuffer`表示。
- `threadCache`是`FastThreadLocal`的子类对象。

这三个属性都在创建`PooledByteBufAllocator`对象时初始化。

看个简单的例子，通过默认的池化分配器，分配一个`1024`字节的ByteBuf内存，

```java
public void testUnpooledDirectByteBuf() {
    PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;
    ByteBuf buffer = allocator.buffer(1024);
    print("allocate ByteBuf(9, 100)", buffer);
}
```

```markdown
PooledDirectBuffer的调用路径
AbstractByteBufAllocator.directBuffer
PooledByteBufAllocator.newDirectBuffer
PoolArena.allocate(PoolThreadCache cache, PooledByteBuf buf, final int reqCapacity)
PoolArena.allocateNormal(PooledByteBuf buf, int reqCapacity, int normCapacity)
PoolArena DirectArena.newChunk(int pageSize, int maxOrder, int pageShifts, int chunkSize)
PoolArena DirectArena.allocate(int capacity)
```

![Desktop View](/assets/images/20260705/AbstractByteBufAllocator_directBuffer.png){: width="800" height="400" }
_AbstractByteBufAllocator的directBuffer方法_

![Desktop View](/assets/images/20260705/PooledByteBufAllocator_newDirectBuffer.png){: width="800" height="400" }
_PooledByteBufAllocator newDirectBuffer_

>形的分配：`newByteBuf`（`Java`对象的对象池`RECYCLER`）<br/>
神的分配：`allocate`（`PoolArena`直接内存池）
{: .prompt-tip }

#### **神的分配 PoolArena.allocate**

![Desktop View](/assets/images/20260705/PoolArena_allocate.png){: width="800" height="400" }
_PoolArena.allocate_

这部分很复杂，分配一块大内存，计算它的块和页，找到应该对应到哪一页，把页的地址拿出来，给到`ByteBuf`就好了。

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity);
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        PoolSubpage<T>[] table;
        boolean tiny = isTiny(normCapacity);
        if (tiny) { // < 512
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }

        final PoolSubpage<T> head = table[tableIdx];

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                incTinySmallAllocation(tiny);
                return;
            }
        }
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
        }

        incTinySmallAllocation(tiny);
        return;
    }
    if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
            ++allocationsNormal;
        }
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        allocateHuge(buf, reqCapacity);
    }
}

// Method must be called inside synchronized(this) { ... } block
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    // Add a new chunk.
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    long handle = c.allocate(normCapacity);
    assert handle > 0;
    c.initBuf(buf, handle, reqCapacity);
    qInit.add(c);
}

@Override
protected PoolChunk<ByteBuffer> newChunk(int pageSize, int maxOrder,
        int pageShifts, int chunkSize) {
    if (directMemoryCacheAlignment == 0) {
        return new PoolChunk<ByteBuffer>(this,
                allocateDirect(chunkSize), pageSize, maxOrder,
                pageShifts, chunkSize, 0);
    }
    final ByteBuffer memory = allocateDirect(chunkSize
            + directMemoryCacheAlignment);
    return new PoolChunk<ByteBuffer>(this, memory, pageSize,
            maxOrder, pageShifts, chunkSize,
            offsetCacheLine(memory));
}

@Override
protected PoolChunk<ByteBuffer> newUnpooledChunk(int capacity) {
    if (directMemoryCacheAlignment == 0) {
        return new PoolChunk<ByteBuffer>(this,
                allocateDirect(capacity), capacity, 0);
    }
    final ByteBuffer memory = allocateDirect(capacity
            + directMemoryCacheAlignment);
    return new PoolChunk<ByteBuffer>(this, memory, capacity,
            offsetCacheLine(memory));
}

private static ByteBuffer allocateDirect(int capacity) {
    return PlatformDependent.useDirectBufferNoCleaner() ?
            PlatformDependent.allocateDirectNoCleaner(capacity) : ByteBuffer.allocateDirect(capacity);
}
```

内存池的内存分配工作，实际上发生在创建块的时候，其他地方只需要做内存地址的计算。

#### **形的分配 PoolArena DirectArena.newByteBuf**

使用了`Java`对象的对象池`RECYCLER`。

![Desktop View](/assets/images/20260705/PoolArena_newByteBuf.png){: width="800" height="400" }
_PoolArena.newByteBuf_

![Desktop View](/assets/images/20260705/PooledUnsafeHeapByteBuf_newUnsafeInstance.png){: width="800" height="400" }
_PooledUnsafeHeapByteBuf.newUnsafeInstance_


