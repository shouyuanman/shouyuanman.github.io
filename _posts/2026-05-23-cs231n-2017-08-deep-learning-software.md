---
title: 深度学习软件
date: 2026-05-23 21:00:00 +0800
categories: [AI, cs231n]
tags: [cs231n, 计算机视觉, 机器学习, 深度学习]
music-id: 1379958635
---

## **回顾上节**

回顾上节课讲过的内容，在上次课中，我们讨论了深度学习的优化算法，包括`SGD`动量、`Nesteroy`、`RMSProp`和`Adam`。我们看到对`SGD`进行微小的微调，这也相对容易实现，便可以使网络更快的收敛。我们也讨论了正则化，尤其是`dropout`，所以要记住`dropout`，它是在正向传播过程中将网络的某些部分随机的设置为`0`，然后在测试时被边缘化。我们发现在各种不同的正则化中，这是一个普遍的模式，在深度学习的训练中加入一些噪声，在测试时再将噪声边缘化，但这就不再是随机的了。我们也讨论了迁移学习，你可以下载一些在大的数据集上预训练过的网络模型，然后根据你自己的实际任务进行微调，这是一种有效方法。当你没有足够大的数据集时，依然可以解决很多深度学习任务。

![Desktop View](/assets/images/20260523/train_nn_recap.png){: width="600" height="300" }
_简单回顾：训练神经网络（来自cs231n）_

## **深度学习框架**

深度学习软件，每年都有新的变化。今天我们讨论一些关于编写软件和硬件，以及它们如何工作的细节，还有一点是，了解一下这个软件的实际应用流程。我们会讨论一些关于`CPUs`和`GPUs`，然后讨论当前几个主流的深度学习框架，比如`Caffe`、`Torch`、`Theano`、`Trensorflow`、`Keras`、`PyTorch`等。

![Desktop View](/assets/images/20260523/deep_learning_software_related.png){: width="300" height="150" }
_深度学习软件相关（来自cs231n）_

### **GPU**

计算机有`CPU`，也有`GPU`，深度学习使用`GPUs`，但我们没有明确的指出它具体是什么，以及为什么它在不同的任务中比`CPUs`表现要好。`CPU`是中央处理器，是一个小芯片。`GPU`像两个巨大的怪物，占据了主机很大的空间，有自己的冷风系统，并需要很大的能量。所以问题是这些东西是什么？它们为什么对于深度学习如此重要？

![Desktop View](/assets/images/20260523/gpus.png){: width="300" height="150" }
_GPU长什么样（来自cs231n）_

`GPU`，被称为图形卡，或者图形处理单元，这些最初是用于渲染计算机图形，特别是围绕游戏或相类似的东西而开发的。

![Desktop View](/assets/images/20260523/gpu_manufacture.png){: width="200" height="100" }
_GPU厂商（来自cs231n）_

`Intel`和`AMD`是`CPU`的竞争，`NVIDIA`和`AMD`是图形卡的竞争，它与`Vim`和`Emacs`的文本编辑器在一起，而且几乎任何一个玩家都有自己的观点。在深度学习中，我们基本上选择了这场战斗的一方，那就是`NVIDIA`。如果你们有`AMD`的卡片，如果你们想要用它们进行深度学习的话，可能会遇到很多麻烦。事实上，在过去的几年里，`NVIDIA`一直在大力推动深度学习，这是他们战略的一大重点。他们投入精力来设计出更好的解决方案，使他们的硬件更适合深度学习，所以当我们深度学习中谈论`GPUs`时，我们完全是在谈论`NVIDIA`的`GPUs`，也许在将来会有新的图像卡产生，但至少现在`NVIDIA`已经占据了主导地位。

`GPUs`和`CPUs`都是一种通用的计算机器，在那里它们可以执行程序和任意指令，但它们在性质上是非常不同的，`CPU`只有为数不多的核数，针对时下的消费者可能会有四个或六个乃至十个核数，利用多线程技术，也就是说，它们可以在硬件上同时运行八个乃至`20`个线程，`CPU`可以同时实现`20`种操作，这并不是一个非常庞大的数字，但`CPU`的线程实际上非常有用，它可以实现很多操作，并且运行速度非常快，每一项`CPU`指令实际上都可以做很多事情，而且它们的运作相互独立。

而`GPU`的情况稍有不同，针对`GPU`我们可以看到，那些高端的消费者`GPU`都有成千上万的核数，`NVIDIA Titan XP`是目前顶级的消费者`GPU`，它有`3840`个核。在差不多的价格下，它比`CPU`还要多`10`个核数，`GPU`有一个缺点是，它的每一个核运行速度非常慢，第二个缺点就是，它们能执行的操作没有`CPU`多，你不能把`CPU`的核跟`GPU`的核进行直接比较，`GPU`的核没办法独立操作，它们需要共同协作，即`GPU`多个核共同执行同一项任务，而不是每个核做各自的事情。你真的不能直接地将数字进行对比，这会给你一种感受，那就是因为`GPU`有着大量的核数，当你需要同时执行多种操作时，`GPU`的并行处理能力是非常棒的，但是那些事情本质上都是相同的。

`CPU`和`GPU`还有一点需要指明，那就是内存的概念。`CPU`有高速缓存，但是相对比较小，而且你拥有的`CPU`的大部分内存都是依赖于系统内存，在一台典型的消费级台式机上`RAM`的容量可能有`8/12/16`或者`32GB`，而`GPU`在芯片中内置了`RAM`，在`RAM`和`GPU`通信时会带来严重的性能瓶颈。`GPU`基本上有它们自己相对较大的内存，而对于`Titan XP`大概是目前最高端的消费级`GPU`，它的本地内存有`12`个`GB`，`GPU`也有它自己的缓存系统，所以就会在`GPU`的`12`个`G`和`GPU`核之间有多级缓存，实际上跟`CPU`的多层缓存有点相似。`CPU`对于通信处理来说是足够的，可以做各种各样的事情，而`GPU`更擅长处理高度并行处理的算法，其中最典型的性能很好同时又完全适用于`GPU`的一个算法，就是矩阵乘法。

![Desktop View](/assets/images/20260523/cpu_vs_gpu.png){: width="600" height="300" }
_CPU vs GPU（来自cs231n）_

在矩阵乘法中，左边是一个由很多行组成的矩阵，乘以右边由很多列组成的矩阵，就可以得到一个乘积的输出矩阵。矩阵的每一个元素都是第一个矩阵的某一行与第二个矩阵的某一列点积的结果。这些点积运算都是独立的。想象一下，对于这个输出矩阵，你可以把它们完全拆分开，将输出矩阵中每个不同的元素并行计算，而所有的计算都是两个向量求点积的运算。但实际上它们是从两个输入矩阵的不同位置读取数据，所以你只需要把矩阵拆开，然后把输出矩阵每个元素并行计算出来。在`GPU`上这可以运行的飞快，所以这是一个典型的适用于`GPU`解决的问题。

换作`CPU`，那`CPU`可能会进行串行计算（串行计算和并行计算相对应），对每个元素一个一个进行计算。说起来有点讽刺，因为今天的`CPU`有多个核，它们也可以实现矢量指令，但是针对那些平行任务，`GPU`可能会表现得更好，特别是当矩阵变得非常庞大时。

![Desktop View](/assets/images/20260523/matrix_multiplication.png){: width="600" height="300" }
_矩阵乘法（来自cs231n）_

顺便说一下，卷积网络也是这样子发展过来的，在卷积运算中有一个输入张量和权重张量，而卷积得到的输出张量的每个点，都是部分权重和部分输入点积的结果。你可以想象，`GPU`真的可以进行并行计算，通过多核将它们完全拆分开，然后进行快速的计算。所以，在这种类型的问题上，`GPU`相对于`CPU`有非常大的运行速度优势。

可以在`GPU`上写出可以直接运行的程序，`NVIDIA`有个叫做`CUDA`的抽象代码，可以让你写出类`C`的代码，并且可以在`GPU`上直接运行，但是写`CPUA`代码不太容易，想要写出高性能，并且充分发挥`GPU`性能的`CUDA`代码很困难，你必须非常谨慎地管理内存结构，并且不遗漏任何一个高速缓存以及分支误预测等等。所以，自己编写高性能的`CUDA`代码很困难。`NVIDIA`开源了很多库，可以用来实现高度优化。`GPU`的通用计算原语，举个例子，`NVIDIA`有个叫做`cuBLAS`的库可以实现各种各样的矩阵乘法，并且那些矩阵操作都是被高度优化的，可以在`GPU`上很好地运行，非常接近硬件利用率的理论峰值。

![Desktop View](/assets/images/20260523/gpu_programming.png){: width="400" height="200" }
_GPU编程（来自cs231n）_

所以，做深度学习时，就不用自己编写`CUDA`代码了。基本上只需要直接调用已经写好的被`NVIDIA`高度优化过的代码。

这里有另一种语言叫做`OoenCL`，这种语言在深度学习中更加普及。它不仅可以在`NVIDIA GPU`上运行，还可以在`AMD`以及`CPU`上运行，但是没有人花费大量的精力优化`OpenCL`的深度学习计算原语，所以`OpenCL`的优化版本在性能上并没有比`CUDA`的好，可能在未来我们会看到更多公开的跨平台的标准，但至少在现在`NVIDIA`会在深度学习中占据绝对的优势。你可以找到各种各样关于`GPU`编程的学习资料，这是个非常有趣的领域。`GPU`的编程遵循另一种范式，因为它有大规模的并行结构，这部分内容超出了本课程的范围，所以实际上你不需要自己编写`CUDA`代码来做深度学习。我就从来没为任何研究项目写过自己的`CUDA`代码。不过计算自己不写代码，知道背后的原理，了解一些基本的概念还是很有用的。

如果想看一下`CPU`和`GPU`实际的性能表现，去年夏天我做了一些基准测试，把一款`Intel`的高端`CPU`同当时性能最好的几款`GPU`做了一下比较，这是我得到的测试结果。更详细的内容可以在`Github`上找到，我发现在`VGG16/19`和不同层数的`ResNet`上，对于完全相同的计算任务，`CPU`耗时是`GPU`的`65~75`倍，这里的`GPU`是旗舰级的`Pascal Titan X`。`CPU`则是顶级，但也不算最顶级的`Intel E5`处理器。不过要提醒一下，在阅读任何这类基准评测时，都要保持非常谨慎的态度，因为在两种事物之间作比较很容易得到不公平的结果，需要了解很多细节，比如这个基准测试到底在测什么，才能判断这样的比较是不是公平的。这里采用直接在`CPU`上安装运行`Torch`，和直接在`GPU`上安装运行`Torch`。我把它们直接拿来就用了，所以测试结果可能不是理论上`CPU`的最高性能。

![Desktop View](/assets/images/20260523/performance_cpu_vs_gpu.png){: width="600" height="300" }
_CPU vs GPU 性能对比（来自cs231n）_

另一个有趣的结果是，比较了`NVIDIA`为卷积优化过的`cuDNN`库，和没有经过优化的发布在开源社区以`CUDA`写成的代码，在相同的网络，相同的硬件，相同的`Deep Learning`框架上，唯一的区别是，`cuDNN`换成了原生手写未经优化的`CUDA`版代码，可以在图表中看到大约有`3`倍的速度差距，也就是说优化过的`cuDNN`比原生`CUDA`版代码快这么多。所以一般来说，只要你在`GPU`上写代码，你就应该使用`cuDNN`，不然根据这个图表你会白白扔掉`3`倍左右的性能增幅。

![Desktop View](/assets/images/20260523/performance_cudnn_vs_cuda.png){: width="600" height="300" }
_cuDNN vs CUDA 性能对比（来自cs231n）_

在实践中，还有一个问题，在训练网络时，模型可能存储在`GPU`上，比如模型的权重存储在这块`12G`大小的`GPU`内存上，但是庞大的数据集却存储在机械硬盘或是固态硬盘上，如果不小心就可能让从硬盘中读取数据的环节成为训练速度的瓶颈，因为`GPU`运行非常快，它计算正向、反向传播的速度非常快，从机械硬盘上串行地读取数据会拖慢训练速度，这会让训练变慢。有一些解决的方法，如果数据集比较小，可以把整个数据集读到内存中，或者哪怕数据集不小，但是你有台插满了内存的服务器，就可以这么干。也可以装一块固态硬盘替换掉机械硬盘，这样读入速度会有很大提升。另一个常用思路是，在`CPU`上使用多线程来预读数据，把数据读出内存或者读出硬盘，存储到内存上，这样就能连续不断的将缓存数据比较高效地送给`GPU`。这种方法不太容易实现，但是因为`GPU`太快了，如果不尽可能快地把数据发送给`GPU`，仅仅读取数据这一项就会拖慢整个训练过程，这是需要注意的一点。

![Desktop View](/assets/images/20260523/cpu_gpu_communication.png){: width="600" height="300" }
_CPU 和 GPU 通信（来自cs231n）_

>Q：在写代码时，怎样才能有效地避免前面提到的那些问题呢？
{: .prompt-tip }

A：从软件上来说，可能你能做到的最有效的事情是设定好`CPU`的预读内容，比如避免这种比较笨的序列化操作，先把数据从硬盘里读出来，等待小批量的数据，一批一批地读完，然后依次送到`GPU`上做正向反向传播，再读下一个小批量的数据，按顺序这样做。如果你有多个比如说多个`CPU`线程，在后台从硬盘中搬运出数据，这样可以把这些过程交错着运行起来，`GPU`在运算的同时，`CPU`的后台线程从硬盘中取数据，主线程等待这些工作完成，在它们之间做一个同步化，让整个流程并行起来。如果使用了下面要讲的这些深度学习框架，它们已经替你完成了这部分操作。

以上是在深度学习的过程中一些关于`GPU`和`CPU`的简单介绍。接下来讨论下软件方面的内容，有各种各样的深度学习框架。

### **框架软件**

![Desktop View](/assets/images/20260523/deep_learning_framework.png){: width="200" height="100" }
_深度学习框架（来自cs231n）_

深度学习框架在飞速发展，去年讲这门课时，主要还是`Caffe`、`Torch`、`Theano`和`Tensorflow`。从去年到今年，`Tensorflow`取得了很大的发展，并且变得流行起来。

![Desktop View](/assets/images/20260523/deep_learning_framework_dev_manufacture.png){: width="300" height="150" }
_各种各样的新框架如雨后春笋般发布出来（来自cs231n）_

现在各种各样的新框架如雨后春笋般发布出来，尤其是`Caffe2`和`PyTorch`这两个来自`Facebook`的新框架，其他框架还有很多，如百度的`Paddle`、微软的`CNTK`等等。今天主要讨论`PyTorch`和`Tensorflow`。

![Desktop View](/assets/images/20260523/deep_learning_framework_dev_manufacture_02.png){: width="400" height="200" }
_各种各样的新框架如雨后春笋般发布出来（来自cs231n）_

从图上可以发现，被广泛使用的第一代深度学习框架大多是由学术界完成的，但下一代深度学习框架全部由工业界产生。

在最近的几次课里，我们已经重复了计算图这种思想很多遍了。无论何时，你进行深度学习，你都要考虑构建一个计算图来计算任何你想要计算的函数。所以，在线性分类器的情况下，你会对数据`X`和权重`W`进行矩阵乘法，你可能会做一些`hinge loss`损失函数来计算你的损失，也会使用一些正则项，你也企图把这些不同的操作拼起来成为一些图结构。

![Desktop View](/assets/images/20260523/computational_graph_recall.png){: width="600" height="300" }
_回顾：计算图（来自cs231n）_

在大型神经网络的情况下，这些图结构会变得非常复杂。现在这有很多不同的层，很多不同的激活函数，很多不同的权重，在这个非常复杂的图的各处。

![Desktop View](/assets/images/20260523/computational_graph_recall_02.png){: width="600" height="300" }
_回顾：计算图（在大型神经网络中）（来自cs231n）_

而且当你转向像神经图灵机这样的东西，那么你可以得到这些非常疯狂的计算图，你甚至不能将其画出来，因为它们实在是太庞大复杂了。

![Desktop View](/assets/images/20260523/computational_graph_recall_03.png){: width="600" height="300" }
_回顾：计算图（在神经图灵机中）（来自cs231n）_

所以，深度学习框架意义重大，有三个主要原因让你会想去使用这些深度学习框架，而不是自己编写代码。

![Desktop View](/assets/images/20260523/the_point_of_deep_learning_frameworks.png){: width="400" height="200" }
_使用这些深度学习框架的三个主要原因（来自cs231n）_

第一，这些框架可以使你非常轻松地构建和使用一个庞大地计算图，而且不用自己去管那些细节的东西。

第二，无论我们何时使用深度学习，我们总是需要计算梯度，我们总是需要计算损失，总是需要计算损失在权重方向的梯度，我们希望能使这些都自动计算梯度，你不会想要去自己写出这些代码，你会希望有个框架能为你处理所有这些反向传播的细节，这样你就可以仅仅考虑如何写出你的网络的前向传播，并且使反向计算直接被给出，无需额外的工作。

第三，你会希望所有这些东西都在`GPU`上高效运行，这样你就不用太关注与一些低级别的硬件上的细节，像`cuBLAS`、`cuDNN`、`CUDA`以及数据在`CPU`和`GPU`内存之间的移动，你会希望所有这些杂乱的细节已经为你处理好了。

作为一个计算图的具体例子，我们也许可以写出这个超级简单的东西，这里有三个输入`XYZ`，我们要结合`X`和`Y`来生成`A`，然后再结合`A`和`Z`来生成`B`，最后我们要对`B`进行求和操作来将一些值传给最终的结果`C`。所以，你可能已经编写了足够多的`Numpy`代码来实现这一点，并且意识到实现这个计算图的代码是非常容易写出来的，或者更确切地说，在`Numpy`中实现这一点计算。你可以在`Numpy`中写下你想要生成的随机数据，想要把这两个东西相乘，想把两个东西相加，想把一些东西求和，这些在`Numpy`中都能轻松做到。

![Desktop View](/assets/images/20260523/computational_graph_numpy_impl.png){: width="600" height="300" }
_使用Numpy实现计算图（来自cs231n）_

但之后问题是，假设我们想要计算`C`相对于`XYZ`的梯度，所以如果是在`Numpy`中进行工作，你可能就需要自己写出反向计算了，对于复杂的计算图来说这就有点麻烦了。

关于`Numpy`的另一个问题是，它不能运行在`GPU`上，所以`Numpy`只能在`CPU`上运行，并且如果你坚持在`Numpy`工作的话，你永远无法体会或者利用这个`GPU`加速对计算进行加速。另外，在这种情况下，必须自己计算梯度是非常痛苦的。所以，这些日子以来，大部分深度学习框架的目标是，让你在前向传播时的代码编写看起来和`Numpy`非常相似，但是又能在`GPU`上运行，并且能自动计算梯度，这就是多数框架的目标。

所以，让我们看一个在`Tensorflow`中完全相同的计算图的例子。现在我们看到，在这个前向传播中，这个代码看起来非常相似于`Numpy`前向传播中的一些乘法和加法的操作。

![Desktop View](/assets/images/20260523/computational_graph_tensorflow_impl.png){: width="600" height="300" }
_使用Tensorflow实现计算图（来自cs231n）_

但是，现在`Tensorflow`有神奇的这么一行替你计算了所有梯度，所以现在你不用自己写反向计算，这样方便太多了。

![Desktop View](/assets/images/20260523/tensorflow_calc_gradient.png){: width="500" height="250" }
_Tensorflow计算梯度（来自cs231n）_

另一个关于`Tensorflow`很棒的地方是，你可以使用这样一行代码来将所有的这些计算在`CPU`和`GPU`之间切换，所以这里如果你添加了这个声明，在进行前向传播之前，你就明确地告诉了框架我想要在`CPU`上运行这些代码。

![Desktop View](/assets/images/20260523/tensorflow_calc_use_cpu.png){: width="600" height="300" }
_Tensorflow使用CPU（来自cs231n）_

但是，我们稍微改变一点点声明，在这个例子中只是改变一个字母，把`C`改成`G`，则代码将会在`GPU`上运行。

![Desktop View](/assets/images/20260523/tensorflow_calc_use_gpu.png){: width="400" height="200" }
_Tensorflow使用GPU（来自cs231n）_

在这个代码小片段里，我们已经解决了这两个问题，我们将我们的代码运行在`GPU`上，并且让框架为我们计算了所有梯度，这真的很棒。

`PyTorch`看起来差不多完全一样，同样在`PyTorch`再次写下你们定义的一些变量，

![Desktop View](/assets/images/20260523/computational_graph_pytorch_impl.png){: width="600" height="300" }
_使用PyTorch实现计算图（来自cs231n）_

以及前向传播，在这个例子中前向传播也非常相似于`Numpy`的代码，

![Desktop View](/assets/images/20260523/pytorch_forward.png){: width="400" height="200" }
_PyTorch前向传播（来自cs231n）_

然而还是一样的，你可以使用PyTorch来计算梯度，所有的梯度计算只需要一行代码。

![Desktop View](/assets/images/20260523/pytorch_calc_gradient.png){: width="400" height="200" }
_PyTorch计算梯度（来自cs231n）_

再次的，在`Pytorch`中切换到`GPU`非常容易，你只需要在运行计算之前，把所有的东西都转换成`CUDA`数据类型，然后一切都在`GPU`上透明地为你运行。

![Desktop View](/assets/images/20260523/pytorch_calc_use_gpu.png){: width="400" height="200" }
_PyTorch使用GPU（来自cs231n）_

所以，如果你只是看这三个例子，这三个并排的代码片段，`Numpy`、`TensorFlow`和`PyTorch`，你们可以看到`TensorFlow`和`PyTorch`的代码在前向传播中和`Numpy`看起来几乎完全一样，因为`Numpy`有一个极好的接口，它非常容易用来一起工作。但是我们可以自动计算梯度，自动运行在`GPU`上。

![Desktop View](/assets/images/20260523/numpy_vs_tensorflow_vs_pytorch_impl.png){: width="600" height="300" }
_Numpy vs Tensorflow vs PyTorch实现（来自cs231n）_

#### **TensorFlow**
`
作为整个讲义的剩余部分的一个运行示例，我将在随机数据上使用一个两层的全连接`ReLU`网络来作为一个运行示例，通过这里剩下的这些代码，并且我们将在随机数据上使用`L2`欧氏距离损失，所以这是个有点蠢的网络，它没有做任何有用的事情，但它确实给你们提供了一个相对来说比较小的一个完整的项目，可以展现框架里面很多有用的思想。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl.png){: width="600" height="300" }
_Tensorflow实现简单的两层全连接神经网络（来自cs231n）_

看右边的代码，假设`Numpy`和`TensorFlow`已经被导入到所有的这些代码中，

![Desktop View](/assets/images/20260523/tensorflow_and_numpy_import.png){: width="300" height="150" }
_导入模块（来自cs231n）_

那么在`TensorFlow`中，通常情况可以把你的计算划分为两个主要阶段。首先我们先用一段代码来定义我们的计算图，就是上半部分，然后可以定义自己的图，这个计算图会运行数次，实际上你可以将数据输入到计算图中去实现任何你想实现的运算，这是`TensorFlow`中非常通用的一种模式。首先用一段代码构建图，然后可以运行图模型，重复利用它很多次。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_part.png){: width="600" height="300" }
_Tensorflow计算实现，划分为两个主要阶段（来自cs231n）_

如果你想深入了解创建图模型的代码，从顶部代码可以看到，我们定义了`X`和`Y`，`w1`和`w2`，并且创建了这些变量的`tf.placeholder`对象，所以这些变量会成为图中的输入节点，这些节点会是图中的入口节点。当我们运行图模型时，会输入数据，将它们放到我们计算图中的输入槽中，实际上这跟内存分配没有一点相似之处，我们只是为计算图建立了输入槽。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_input.png){: width="600" height="300" }
_Tensorflow为计算图建立输入槽（来自cs231n）_

然后，我们用这些输入槽，就是这些符号变量，在这些符号变量上执行各种`TensorFlow`操作，以便构建我们想要运行在这些变量上的计算。因此，基于这种情况我们做了一个矩阵乘法，使用`tf.maximum`来实现`ReLU`的非线性特性，然后用另一个矩阵乘法来计算我们输出的预测结果，然后又用基本的张量计算来计算欧氏距离，以及计算目标值`Y`和预测值之间的`L2`损失。这里需要指出的是，这几行代码没有做任何实质上的运算，目前系统里还没有任何数据，我们只是建立计算图数据结构来告诉`TensorFlow`当输入真实数据时，我们希望最终执行什么操作，因此这里只是建立图模型，并没有做任何操作。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_build_model.png){: width="600" height="300" }
_Tensorflow建立图模型（来自cs231n）_

然后在完成损失值的符号运算之后，这里有一行神奇的代码，接下来可以让`TensorFlow`去计算损失值在`w1`和`w2`方向上的梯度，通过这一行神奇的代码，可以免去编写作业中那种反向传播代码的麻烦。但再次强调，这里没有进行实际的计算，它只是在计算图中加入额外的操作，通过这些操作，计算图可以为你算出梯度。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_add_gradient.png){: width="600" height="300" }
_Tensorflow在计算图中加入计算梯度的操作（来自cs231n）_

这里，我们已经完成了计算图的运算，内存中存储了计算图数据结构的大量数据，它包含了为了要计算损失的梯度，我们希望做哪些操作的信息。现在我们进入一个`TensorFlow`的会话，来实际运行计算图并且输入数据。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_session.png){: width="600" height="300" }
_进入一个TensorFlow的会话（来自cs231n）_

一旦我们进入会话，就需要建立具体的数据来输入给计算图，所以大多数时候，`TensorFlow`只是从`Numpy`数组接受数据，这里我们只是使用`Numpy`为`X/Y/w1/w2`创建具体的数值，并在字典中进行存储。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_fill_vector_data.png){: width="600" height="300" }
_建立具体的数据来输入给计算图（来自cs231n）_

这时，我们才是真正在运行计算图，

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_do_run.png){: width="600" height="300" }
_运行计算图（来自cs231n）_

你可以看到，我们调用了`session.run`来执行部分计算图的运算，第一个参数`loss`告诉我们，图的哪一部分作为实际的输出，我们实际希望的图。在这个例子中，我们需要告诉它，我们希望计算`loss`、`grad_w1`和`grad_w2`，我们需要通过字典参数进行传递，这些具体的数值将会输入到计算图中，然后通过一行代码来运行计算图，并且计算`loss`、`grad_w1`、`grad_w2`这些值，然后向`Numpy`数组再次返回具体的数值。在代码的第二行，将输出解包，你将得到`Numpy`数组，或者包含损失值和梯度的`Numpy`数组，根据这些值你可以进行任何操作。

至此，我们只是在计算图上进行了正向传播和反向传播运算，如果想训练网络需要添加几行代码。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn.png){: width="600" height="300" }
_训练网络的代码部分（来自cs231n）_

目前，我们在循环中运行了多次计算图进行了四个循环，在循环的每一次迭代中调用`session.run`请求`TensorFlow`去计算损失和梯度，现在我们可以使用计算得到的梯度更新当前的权重值来完成一步手工梯度下降。如果你运行这个代码，并画出损失值曲线，就会发现损失值越来越小，网络的训练非常好。这就是一个在`TensorFlow`中进行全连接网络训练最基本的例子。

但在这里有一个问题，当我们每次执行计算图进行前向传播时，我们实际上在对权重进行输入，我们将权重保存成`Numpy`的形式，我们直接将它输入到计算图，当计算图执行完毕，我们会得到这些梯度值。记住，这些梯度值的个数与权重的个数一致，这意味着我们每次运行图的时候，我们将从`Numpy`数组中复制权重到`TensorFlow`中才能得到梯度，然后从`TensorFlow`中复制梯度到`Numpy`数组。如果你只是在`CPU`上运行它，这可能不是个大问题。但是我们曾谈到`CPU`和`GPU`之间的传输瓶颈，要在`CPU`和`GPU`之间进行数据复制非常耗费资源，所以如果你的网络非常大，权重值和梯度值非常多，这样做就很耗费资源，并且很慢，因为每个时钟周期它都在`CPU`和`GPU`之间来回拷贝各种数据，那是非常糟糕的，我们不想那样做，我们得修正它。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_copy_problem.png){: width="600" height="300" }
_训练网络的代码部分，存在copy数据的传输瓶颈（来自cs231n）_

很明显`TensorFlow`已经有了解决方案，这个想法就是将权重`w1`和`w2`定义为变量，而不是在每次前向传播时都将它们作为需要输入网络的占位符，变量可以存在计算图中，并且当你在不同时间运行相同计算图时，它都可以保持在计算图中，所以我们用创建变量来代替将`w1`和`w2`声明为占位符，但正因为它们存在于计算图中，我们需要告诉`TensorFlow`如何对它们进行初始化。在前一个例子中，我们从计算图外部输入这些值，所以我们在`Numpy`中进行初始化，但是现在它们存在于计算图中，因此`TensorFlow`负责初始化这些值，所以我们需要执行`tf.randomnormal`操作，同样也不是真正的初始化它们。当运行这行代码时，它只是告诉`TensorFlow`我们希望怎样进行初始化，在这里容易被误导。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_init_weight.png){: width="600" height="300" }
_告诉TensorFlow如何对权重进行初始化（来自cs231n）_

现在记住在上一个例子中，我们实际上是在计算图之外对权重值进行的更新操作。在之前的例子中，我们计算梯度，然后以`Numpy array`的形式更新权值参数，然后，在下一次迭代时利用这些更新过的权重参数。但是现在我们希望在计算图中操作，更新参数的操作也需要成为计算图中的一个操作，所以现在我们利用这个赋值函数，其能在计算图中改变参数值，然后这些变化的值会在计算图的多次迭代之后仍然保存。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_assign_update.png){: width="600" height="300" }
_更新参数的操作也需要成为计算图中的一个操作（来自cs231n）_

当我们执行这个计算图训练这个网络时，需要首先进行一个全局参数的初始化操作，告诉`TensorFlow`来设置计算图中的这些参数。一旦我们初始化完成之后，我们可以一次又一次地运行计算图，这里我们只输入了数据`X`和标签`Y`、计算图中的权重参数，我们要求`TensorFlow`帮我们计算`loss`。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_init_global_param.png){: width="600" height="300" }
_全局参数的初始化（来自cs231n）_

这里你可能认为这已经可以训练网络了，但这里有一个`bug`。如果你运行这个代码，把`loss`打印出来，它并没有进行训练，这让人费解，到底发生了什么？

我们写了赋值代码，执行了这些代码，正如我们计算了`loss`和梯度值，我们的`loss`值是平的并没有下降，发什么什么事？

一种可能是我们每次运行计算图时都进行了初始化操作，这是一个非常好的假设，但是这个`bug`不是这样的问题，答案是我们必须明确地告诉`TensorFlow`我们想要利用`new w1`和`new w2`来执行操作，我们在显存中建立了这么大的计算图架构，当我们执行操作时，只告诉`TensorFlow`我们想计算`loss`，如果你仔细观察图中这些不同的操作，你会发现为了计算`loss`，我们并不需要去执行更新操作，`TensorFlow`太聪明，它只执行了你要求结果所必须的操作，这是非常棒的，因为这意味着它只做它需要做的工作，但是在上面这个情况，它会让人产生疑惑，产生了一种你意想不到的行为。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_loss_not_down.png){: width="600" height="300" }
_loss值是平的，并没有下降（来自cs231n）_

所以，这个问题的解决方案是，我们必须明确地告诉`TensorFlow`来执行更新操作，我们需要做的一件事，也是我们推荐的，我们应该添加`new w1`和`new w2`作为输出，告诉`TensorFlow`我们想要这些结果，但这里还有一个问题，这些`new_w1`和`new_w2`都是非常大的`tensor`，当我们告诉`TensorFlow`我们需要这些输出时，我们在每次迭代中就会在`CPU`和`GPU`之间执行这些操作，这太糟糕了，我们不希望这样，事实上我们在图中添加一个仿制节点，伴随这些仿制数据的独立性，我们就可以说，仿制节点的更新拥有了`new_w1`和`new_w2`的数据依赖性，现在当我们执行计算图时，我们同时计算`loss`和这个仿制节点，这个仿制节点并不返回任何值，因为我们放入节点这个数据依赖，保证了当我们执行了更新操作后我们使用了更新的权重参数值。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_add_dummy_graph_node.png){: width="600" height="300" }
_在图中添加一个仿制节点（来自cs231n）_

>Q：为什么我们不把`X`和`Y`放入计算图中，为什么它必须是`Numpy`这种数据结构？
{: .prompt-tip }

A：在这个例子中，我们在每个迭代都重复使用了同样`X`和`Y`，所以我们本可以把这些放入计算图中，但在更加现实情况里，`X`和`Y`是数据集的`mini batch`，所以他们在每次迭代时都是变化的，这样我们想要在每次迭代都要输入不同的值，在我们这种情况下，他们可以放在图里，但大多数情况下他们会改变，所以我们不希望他们放在图中。

>Q：我们之前说的，我们想要的输出是`loss`和更新标志，更新标志并不是一个真的数值，更新标志返回是空，但是因为这个依赖意思是更新依赖于这些赋值操作，但是这些赋值操作在计算图里面都存储于`GPU`显存中，所以我们在`GPU`中执行这些更新操作，而不需要把更新数值从图中拷贝出来。
{: .prompt-tip }

>Q：`tf.group`返回空，这个说到了`TensorFlow`的小诡计，`tf.group`返回了一些相当疯狂的值，在某种方式上它返回了`TensorFlow`的内部节点操作，我们需要这些节点操作来构建图，但是当我们执行图时，在`session.run`操作里，我们告诉它我们想要从更新中计算具体值，然后它返回空。不论你什么时候在`TensorFlow`中操作，你会碰到这种可笑的间接操作，在构建图和构建图时的真实输出值之间，你实际上会得到一个具体值当你执行计算图时，在你执行更新之后，输出就是空。
{: .prompt-tip }

>Q：为什么`loss`是一个值？为什么更新是空？
{: .prompt-tip }

A：这只是更新操作的方法，`loss`是我们计算的一个值，当我们告诉`TensorFlow`我们想要计算一个`tensor`时，然后我们得到一个具体值，更新可以看成一种特殊的数据类型，它并不返回值，相反它返回空，所以你可以把这里当做`TensorFlow`的魔力。

我们有一种奇怪的模式，在我们想要执行不同赋值操作的地方，我们需要利用`tf.group`，这其实是有点痛苦的，还好`TensorFlow`给了我们便捷操作来做这些事情。这里我们使用了`tf.train.GradientDescentOptimizer`函数，我们传入学习率这个参数值，你可以看到这里是`RMSProp`，这里有很多不同的优化算法，我们现在调用`optimizer.minimize`来最小化我们的损失函数，这是一个非常有用的函数，因为通过这个调用可以知道，这些变量`w1`和`w2`在默认情况下被标记为可训练，因此在这个`optimizer.minimize`里面，它会进入计算图并在计算图中添加节点，计算关于`w1`和`w2`的损失梯度，然后它执行更新操作，以及进行分组操作，还有进行分配。这就像在里面进行了很多神奇的过程，但是最后终止时会给你一个神奇的更新值。这如果你仔细查看代码，他们实际上使用了`tf.group`，所以它从内部看起来和我们之前看到的非常相似，当我们在循环中运行计算图时，我们采用相同的模式来计算损失值和更新值，每次我们仍让计算图进行更新，它就会进行计算并更新计算图。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_optimizer.png){: width="600" height="300" }
_optimizer（来自cs231n）_

在使用我们自己的张量来精确的计算损失，你总是可以用`TensorFlow`做到这一点，你可以使用基本的张量操作去计算你想要的任何东西，但是`TensorFlow`也给了我们很多方便的函数，它们能帮你计算一些常见的神经网络的相关结果，所以在这个例子中，我们可以使用`tf.losses.mean_squared_error`，它只是为我们计算`L2`损失，所以我们没有用基本的张量操作来计算它。另一个奇怪的并令人烦恼的地方是，我们必须明确定义我们的输入，还有定义我们的权重，然后像使用矩阵乘法似的，在正向传播中将它们链接在一起，而这个例子中，我们其实没有在网络层中放入偏差，因为偏差算是额外的部分，所以我们必须初始化偏差，我们得让它们保持正确的形式，我们不得不传播矩阵乘法的输出偏差，你可以看到这里有很多的代码，这是一种令人反感的写法。一旦你喜欢上了卷积层、批量规范层和其他类型的层，这种基本的工作方式有这些变量，有这些输入和输出，然后用基本的计算图操作把它们结合起来，这种方式可能会有点麻烦。可能会更烦人的一件事是，你在初始化权重时，你得确保你用了正确的形状和其他正确的东西。在`TensorFlow`中有一堆高级库可以为你处理这些细节。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_mean_squared_error.png){: width="600" height="300" }
_mean_squared_error（来自cs231n）_

`TensorFlow`附带的一个例子是`tf.layers`，因此在这个代码实例中，你可以看到我们的代码只是显式说明了`x`和`y`，它们是数据和标签的占位符。现在我们使用`H=tf.layers.dense`，我们把`x`作为输入，单元数为`H`，这又是很神奇的一行，因为在一行里，它设置了`w1`和`b1`，也就是偏差。它为那些形状正确的设置变量，这些变量存在于计算图中，但对我们来说，可以说是隐藏的。它使用`Xavie initialize`来为这些变量建立一个初始化策略，所以在以前，我们明确地使用`tf.randomnormal`来做这些事，但是在这里的这个，帮我们处理了一些细节，而且只是输出了一个`h`，这个`h`又是我们在前面看到的那种层，所以它只是为我们进行了一些细节上的处理。而且在这里你可以看到我们也传入了`activation=tf.nn.relu`这个激活函数，也就是这个层里的激活函数时`ReLU`函数，所以它帮我们注意到了很多的结构上的细节。

![Desktop View](/assets/images/20260523/tensorflow_nn_impl_train_nn_layer.png){: width="600" height="300" }
_layer（来自cs231n）_

`tf.contrib.layer`并不是这里唯一的方式，人们在`TensorFlow`之上建立了许多不同的高级库，这是由于这种基本的无能为力的不匹配，其中计算图是相对较低水平的事物，但是当我们正在使用神经网络时，我们有这个层和权重的概念，还有一些层的权重与他们相关联，比起这个原始的计算图，我们通常考虑的是稍高的层次，所以这就是这些各种包可以帮助你的地方。可以让你在更高抽象层次上进行工作。

所以，另一个很受欢迎的包——`Keras`，`Keras`是一个非常方便的`API`，它建立在`TensorFlow`的基础之上，并且在后端为你处理那些你建立的计算图，顺便说一句，`Keras`也支持`Theano`作为后端。

#### **Keras**

![Desktop View](/assets/images/20260523/keras.png){: width="600" height="300" }
_Keras（来自cs231n）_

在上面这个例子中，你可以看到我们构建了一个序列层的模型，

![Desktop View](/assets/images/20260523/keras_build_model.png){: width="600" height="300" }
_Keras构建模型（来自cs231n）_

我们构建了一些优化器对象，

![Desktop View](/assets/images/20260523/keras_build_optimizer.png){: width="600" height="300" }
_Keras构建优化器（来自cs231n）_

然后调用`model.compile`，这里用了很多魔术，后端建立了计算图，

![Desktop View](/assets/images/20260523/keras_call_compile.png){: width="600" height="300" }
_Keras调用model.compile（来自cs231n）_

现在我们可以调用`model.fit`，它自动的就为我们完成了整个训练过程。所以我不知道这些具体是怎么工作的，但我知道`Keras`的使用率很高，所以如果你在用`TensorFlow`，那么你可以考虑使用`Keras`。

![Desktop View](/assets/images/20260523/keras_call_fit.png){: width="600" height="300" }
_Keras调用model.fit（来自cs231n）_

实际上这就像一套更高层次的`TensorFlow`封装，你可能会在其它地方看到，而且`Google`内部的人似乎也不能确定哪一个是正确的，`Keras`和`TFLearn`是第三方库，它们是其他人在互联网上发布的，这里有三种不同的方案，它们都是`TensorFlow`里面的在这个高级包装程序中，它们的功能各不相同。还有一个框架来自`Google`，但不在`TensorFlow`的框架内，它叫做`Pretty Tensor`，它也能做同样的事情。我猜这些对`DeepMind`来说都不够好，因为他们开始几个星期，编写并发布了一个高级`TensorFlow`封装，叫做`Sonnet`。这里有很多的选择，它们相互间并不能很好地兼容。但选择多了也不是坏事。

![Desktop View](/assets/images/20260523/tensorflow_other_high_level_wrappers.png){: width="600" height="300" }
_其他高级包装程序（来自cs231n）_

`TensorFlow`是有预训练模型的，这里有`TF-Slim`和`Keras`的一些例子。记住在训练自己的任务时，重新训练模型很重要。

![Desktop View](/assets/images/20260523/tensorflow_pretrained_models.png){: width="600" height="300" }
_TensorFlow预训练模型（来自cs231n）_

还有一个知识点，就是`TensorBoard`，你可以添加一些指示性的代码，从而使用`TensorBoard`画出训练过程中的`loss`曲线和一些其他内容。

![Desktop View](/assets/images/20260523/tensorflow_tensorboard.png){: width="600" height="300" }
_TensorBoard（来自cs231n）_

`TensorFlow`也可以实现分布式运行，从而在不同的机器上进行运算。这很酷，但现阶段只有`Google`可以善用这项特性。如果你是在想使用分布式，`TensorFlow`可能是最合适的选择。

![Desktop View](/assets/images/20260523/tensorflow_distributed_version.png){: width="600" height="300" }
_TensorFlow分布式运行（来自cs231n）_

#### **Theano**

值得一提的是，`TensorFlow`的多数设计都受到一个早期架构的启发，即`Theano`。如果看代码，会发现`Theano`和`TensorFlow`非常相似。我们定义一些变量，进行前向传播，计算梯度，然后编译运行函数，一遍又一遍的训练网络。

![Desktop View](/assets/images/20260523/theano.png){: width="600" height="300" }
_Theano（来自cs231n）_

![Desktop View](/assets/images/20260523/theano_02.png){: width="600" height="300" }
_Theano 运行function函数来训练（来自cs231n）_

#### **PyTorch**

`Facebook`的`PyTorch`不同于`TensorFlow`，`PyTorch`内部明确定义了三层抽象。`PyTorch`的张量对象就像`Numpy`数组，它只是一种最基本的数组，与深度学习无关，但可以在`GPU`上运行。我们有变量对象，就是计算图中的节点，这些节点构成了我们的计算图，从而可以计算梯度等等。我们也有模对象，它是一个神经网络层，可以将这些模组合起来建立一个大的网络。

![Desktop View](/assets/images/20260523/pytorch_three_levels_abstract.png){: width="400" height="200" }
_PyTorch三层抽象（来自cs231n）_

来大致看一下`PyTorch`和`TensorFlow`的对应关系，我们可以将`PyTorch`中的张量视为`TensorFlow`中的`Numpy array`。`PyTorch`中的变量与`TensorFlow`的张量变量或占位符相似，它们在计算图中都算一个节点。`PyTorch`的模可以等价为`tf.slim`、`tf.layers`或者`sonnet`或者其他更高层次架构。`PyTorch`有一点需要注意的是，因为它的抽象层次很高，而且含有像`Module`这样好用的高层抽象模块，这样你选择的余地就变小了，使用`nn.Module`就能得到很好的结果，你不需要担心要使用哪些更高层的封装。

就像之前说的，`PyTorch`中的张量就像`Numpy arrays`，在右边我们只使用`PyTorch`张量实现了一个完整的两层神经网络。要注意的一点是，在这里我们不需要再导入`Numpy`，我们只用`PyTorch`的张量就完成了所有操作。这个代码看上去就像第一次作业中用`Numpy`写的两层网络的代码，先建立一些随机数据，使用一些操作进行前向传播，然后我们清楚地看到按步骤进行的反向传播，最后用学习率和计算出的梯度手动更新权值。

![Desktop View](/assets/images/20260523/pytorch_tensor.png){: width="600" height="300" }
_PyTorch 张量（来自cs231n）_

`PyTorch`张量和`Numpy arrays`之间最大的差异在于张量在`GPU`上的运行，要让这些代码运行在`GPU`上，你需要使用一种不同的数据类型，不再是`torch.FloatTensor`，而应该是`torch.cuda.FloatTensor`，把所有的张量转换为新的数据类型，在`GPU`上都可以很好地运行，你们应该把`PyTorch`张量看成是`Numpy`加上`GPU`，`PyTorch`的张量就是这么回事，并不是只能用在深度学习上。

`PyTorch`中下一层抽象就是变量，一旦我们从张量转到变量，我们就建立了计算图，可以自动做梯度和其他的计算。像这里，如果`X`是一个变量，`x.data`就是一个张量，`x.grad`就是另一个变量，包含了损失对张量`x`的梯度，`x.grad.data`则是一个含这些梯度的实际的张量。

![Desktop View](/assets/images/20260523/pytorch_autograd.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

`PyTorch`张量和变量有相同的`API`，任何在`PyTorch`张量上可行的代码都可以用变量替换，而且运行同样的代码，现在你建立了一个计算图，而不仅仅是做这些命令式的操作。

![Desktop View](/assets/images/20260523/pytorch_autograd_02.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

当我们建立了这些变量，每一次对变量的构造器的调用都封装了一个`PyTorch`张量，并设置了一个二值的标记，告诉构造器我们需不需要计算在该变量上的梯度。

![Desktop View](/assets/images/20260523/pytorch_autograd_03.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

现在的前向传播，与之前使用张量`Tensor`的实现完全一样，因为`API`是相同的。现在来计算一下预测值，我们用这种命令的方式计算损失。

![Desktop View](/assets/images/20260523/pytorch_autograd_04.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

然后调用`loss.backwards()`，就可以得到我们需要的所有梯度值。

![Desktop View](/assets/images/20260523/pytorch_autograd_05.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

然后，我们可以用梯度对权重进行更新，这些梯度都保存在`w1.grad.data`中。这个看上去很像`Numpy`的方式，除了梯度都是自动求解的这一点。

![Desktop View](/assets/images/20260523/pytorch_autograd_06.png){: width="600" height="300" }
_PyTorch 自动做梯度计算（来自cs231n）_

需要注意的一点是，`PyTorch`和`TensorFlow`有一点不同，就是在`TensorFlow`中我们先构建显式的图，然后重复运行它，而在`PyTorch`中，我们在每次做前向传播时都要构建一个新的图，这使程序看上去更加简洁一点。

在`PyTorch`中，你可以自己定义张量的前向和后向来构造新的`autograd`函数，下面的代码看上去像是作业`2`中的模型层代码，你可以在里面使用张量进行前向和反向操作，然后把这些操作固定在计算图中。

![Desktop View](/assets/images/20260523/pytorch_new_autograd_function.png){: width="400" height="200" }
_PyTorch 自己定义张量的前向和后向来构造新的autograd函数（来自cs231n）_

我们来定义一下`ReLU`，然后深入并使用这个`ReLU`，把它固定在计算图中，这样就定义了我们自己的运算，但大多数时候，你可能并不需要定义自己的`autograd`，通常情况下你需要的这些操作都已经准备好了。

![Desktop View](/assets/images/20260523/pytorch_new_autograd_function_02.png){: width="600" height="300" }
_PyTorch 自己定义张量的前向和后向来构造新的autograd函数（来自cs231n）_

在之前看到的`TensorFlow`中，如果迁移到`Keras`或者`TF.Learn`，我们获得了更高层次的`API`，而不是原始的计算图，在`PyTorch`中等价的就是`nn`包，它提供了这些操作对应的高级封装，不像`TensorFlow`有很多高级封装，`PyTorch`只有这一个，所以`PyTorch`只用这一个封装器就足够了。

![Desktop View](/assets/images/20260523/pytorch_nn.png){: width="600" height="300" }
_PyTorch nn（来自cs231n）_

它看上去就像`Keras`，我们把模型定义为一些层的序列、线性层和`ReLU`操作，`nn`中定义了一些损失函数就是我们的均方差损失。

![Desktop View](/assets/images/20260523/pytorch_nn_02.png){: width="600" height="300" }
_PyTorch nn（来自cs231n）_

循环体中每一次迭代时，我们都可以在模型中前向传送数据，得到预测值，把预测值放入损失函数，得到损失值，然后调用`loss.backward`自动计算所有梯度，然后在模型的所有参数上循环，进行显式的梯度下降操作来更新模型。

![Desktop View](/assets/images/20260523/pytorch_forward_pass.png){: width="500" height="250" }
_PyTorch forward pass（来自cs231n）_

我们又一次看到，每次做前向传播时，都建立了一张新的计算图。与`TensorFlow`一样，`PyTorch`提供了优化操作，将参数更新的流程抽象出来，并执行`Adam`之类的更新法则。这里我们建立了一个`optimizer`对象，告诉它我们想要对模型中的参数进行优化，把学习率之类的超参数设置给它。

![Desktop View](/assets/images/20260523/pytorch_optimizer.png){: width="600" height="300" }
_PyTorch optimizer（来自cs231n）_

并在计算了梯度之后，我们就可以调用`optimizer.step`来更新模型中所有的参数。

![Desktop View](/assets/images/20260523/pytorch_optimizer_step.png){: width="500" height="250" }
_PyTorch 更新模型中所有的参数（来自cs231n）_

`PyTorch`中还有一件通常要做的事，就是定义自己的`nn`模块。通常要写出自己的类，这个类把整个模型定义成`nn`模块中的一个新的类，一个模块只是神经网络的一种层，它可以包含其他的模块或者可训练的权重或其他状态。

![Desktop View](/assets/images/20260523/pytorch_nn_define_new_module.png){: width="600" height="300" }
_PyTorch 定义自己的nn模块（来自cs231n）_

例子中，我们通过定义自己的`nn`模块类重现这两层网络。在类的初始化操作时，我们分配了`linear1`和`linear2`，建立新的模块对象，然后存在类中，现在在前向传播时，我们可以使用自己的内部模块，也可以对变量使用任意`autograd`操作来计算网络的输出，这里是`forward`方法的内容作为输入的变量，然后把输入的变量传给`self.linear1`作为第一层，我们用`autograd`操作`clamp`函数计算`relu`，再把输出传给`linear2`，然后我们就得到输出`y_pred`。

![Desktop View](/assets/images/20260523/pytorch_nn_define_new_module_02.png){: width="500" height="250" }
_PyTorch 定义自己的nn模块（来自cs231n）_

那么这段训练的代码就基本上看起来一样了，我们建立了优化器，然后不断地循环，在每次迭代中喂数据给模型，通过`loss.backwards`来计算梯度，调用`optimizer.step`，所以这是比较典型的。

![Desktop View](/assets/images/20260523/pytorch_nn_build_and_train.png){: width="500" height="250" }
_PyTorch 训练（来自cs231n）_

在你看到很多的`PyTorch`训练场景中，你可以定义你自己的类，定义你自己的模型包括其他的模块，然后你可以进行明确的训练，运行它然后更新它。

在`PyTorch`里面有一个比较好的东西，叫`dataloader`。一个`dataloader`可以让你建立分批处理，它也可以执行我们刚刚提到的多线程，事实上它可以用多线程在后台来建立很多批处理和硬盘加载。所以`dataloader`可以打包数据，给你提供一些抽象。在这个示例中当你需要执行你自己的数据时，你会编写自己的数据集类，可以来读取你特殊类型的数据，从你想要的来源然后在`dataloader`里打包并且训练它们。

![Desktop View](/assets/images/20260523/pytorch_dataloaders.png){: width="600" height="300" }
_PyTorch dataloader（来自cs231n）_

所以，在这里我们可以看到，我们迭代`dataloader`对象，在每次迭代的过程中都可以产生分批的数据，然后在其内部重排数据，多线程加载数据，这些都是你的东西。

![Desktop View](/assets/images/20260523/pytorch_dataloaders_02.png){: width="600" height="300" }
_PyTorch dataloader（来自cs231n）_

所以，这就是一个完整的`PyTorch`例子，很多`PyTorch`训练数据结束时看起来都是这样的。

`PyTorch`提供一些预先训练好的模型，这大概是我见过的最轻松的预先训练模型试验，你只要让`orchvision.models.alexnet` `pertained=true`，然后让它在后台跑，下载预先训练。好的权重如果你还没获得，然后你就可以开始训练了。

![Desktop View](/assets/images/20260523/pytorch_pertained_model.png){: width="600" height="300" }
_PyTorch pertained model（来自cs231n）_

`PyTorch`同样有一个包叫`Visdom`，可以让你可视化很多损失统计，类似于`TensorBoard`。它和`TensorBoard`有一个主要的不同是，`TensorBoard`可以让你可视化计算图的结构，这非常有用，是一个`debug`很好的工具，但`Visdom`目前还没有这个功能。

![Desktop View](/assets/images/20260523/pytorch_visdom.png){: width="600" height="300" }
_PyTorch Visdom（来自cs231n）_

顺便说一句，`PyTorch`是过去一种旧框架的更新，叫`Torch`。

![Desktop View](/assets/images/20260523/pytorch_torch.png){: width="600" height="300" }
_Torch（来自cs231n）_

![Desktop View](/assets/images/20260523/pytorch_vs_torch.png){: width="400" height="200" }
_Torch vs PyTorch（来自cs231n）_

下面说一下静态和动态图，这是`PyTorch`和`TensorFlow`最大的不同。所以我们看`TensorFlow`有两个操作的阶段，这里第一步你建立了计算图，然后运行一遍又一遍，运用同一个图，我们把这个叫做静态计算图，因为只用它们中的一个。然后我们可以看`PyTorch`比较不同，当我们建立一个新的计算图时，它会每一次前向传播都更新，我们把它称作动态计算图。

对于这些简单的例子都是前馈神经网络，它们并没有很大的区别，代码看起来都差不多，运作看起来也差不多，但我还是想讲一下其中关于静态和动态的含义，我们该如何从中权衡。

![Desktop View](/assets/images/20260523/static_or_dynamic_graph.png){: width="600" height="300" }
_静态和动态图（来自cs231n）_

其中一个关于静态图的点，是我们只会建一次，然后不断地复用它，然后整个框架会有机会在图上做优化，结合一些操作，排列一些操作，可以找到更有效率的操作方法，所以图可以更加有效率。因为我们可以复用图很多次，可能的优化过程会在前段代价比较高，但我们可以通过获得的加速来分摊成本。当我们复用这个图很多次，这是一个具体的例子。

![Desktop View](/assets/images/20260523/static_or_dynamic_graph_optimization.png){: width="400" height="200" }
_优化过程会在前段代价比较高，可以通过获得的加速来分摊成本（来自cs231n）_

也许你可以写一些有卷积的图，和`ReLU`操作一个接一个，你大概可以想象一些花哨的图优化器，可以传入并产出类似输出自定义代码可以有结合的操作。结合卷积核`ReLU`，所以现在就能计算出相同的东西，和你写的代码一样，不过现在可能可以更有效率的执行，所以我不是非常确切。目前`TensorFlow`的实际使用图优化情况，但至少在原则上，这是一个静态图，你可以有机会在静态图上优化，但对于动态图上可能没那么容易。

另外一个关于静态和动态的点是，执行序列化。你可以想象对于静态图来说，你写的代码对于建立图来说，你一旦建了一次图，在内存中就有了这数据结构，代表你的网络中整个结构，然后现在你可以用这个数据结构在磁盘中序列化，然后你有了整个网络的结构，就可以存在文件中。然后你可以等一下再加载它们，然后运行计算图而不是再去访问最早的代码去建立它。所以在一些部署方案中比较实用。你会想在`Python`中训练你的网络，因为运行简单容易。但你在序列化网络后，可以把它部署到`C++`环境中，你就没必要去重新用原先的代码来建立图，所以这是静态图相对于动态图的一个优势。

![Desktop View](/assets/images/20260523/static_or_dynamic_graph_serialization.png){: width="400" height="200" }
_执行序列化（来自cs231n）_

因为我们是交叉执行图的建立和执行，所以如果你未来想要复用模型，总是需要原来的代码。

另一方面，动态图的优势是，它让你的代码在一些场景看起来更简洁。例如，假设我们想做些条件运算，根据变量`z`的值，我们想做不同的操作来计算`Y`。如果`Z`是正的，我们会使用一个权重矩阵，如果`Z`是负的，我们会用另一种权重矩阵，我们想在这两个选项中切换，因为在`PyTorch`中我们使用动态图，这就非常简单。你的代码看起来跟你期待的一样，你可以用`Numpy`来解决，你可以只用正常的`Python`控制流来操控这个事。

![Desktop View](/assets/images/20260523/static_or_dynamic_graph_conditional.png){: width="600" height="300" }
_条件运算（来自cs231n）_

因为你每次都要建图，每次我们都会执行这个操作来选择不同的条件来建立一个不同的图，在每次前向传播中，当我们结束建立，我们还可以反向传播也是可以的。所以这段代码非常简洁，也很容易。但在`TensorFlow`的情况下就有点复杂了，因为我们只会建一次图，这样的控制流操作就需要一个明确的操作在`TensorFlow`图中，所以你可以看到我们有`tr.cond`在`TensorFlow`中就像是`if`语句，但它被放到计算图中而不是`Python`那样的控制流中，所以问题就在于因为我们只会建立图一次，所有的可能的控制流路径我们都需要提前建立好，放到图的函数中，在我们运行之前。所以这就意味着任何控制流的操作，需要不是`Python`控制流操作，你需要使用一些特殊的`TensorFlow`流操作来控制。在这个例子中，是`tf.cond`。

![Desktop View](/assets/images/20260523/static_or_dynamic_graph_loop.png){: width="400" height="200" }
_loop（来自cs231n）_

![Desktop View](/assets/images/20260523/static_or_dynamic_graph_loop_02.png){: width="600" height="300" }
_loop 实现对比（来自cs231n）_

![Desktop View](/assets/images/20260523/dynamic_graph_in_tensorflow.png){: width="400" height="200" }
_Tensorflow中的动态图（来自cs231n）_

![Desktop View](/assets/images/20260523/dynamic_graph_application.png){: width="400" height="200" }
_动态图应用（来自cs231n）_

为什么我们要关注动态图？

其中一个原因是循环网络，例如图像描述，我们使用循环网络在一个不同长度的序列上运行。在这个例子中，要生成的用来描述的句子是一个序列，一个依赖于输入数据的序列。你可以看出动态取决于句子的大小。我们计算图可能需要一些元素，这是动态图的一个常用的应用。

![Desktop View](/assets/images/20260523/dynamic_graph_application_02.png){: width="400" height="200" }
_动态图应用（来自cs231n）_

循环神经网络常用与自然语言处理，举个例子，你可能想要计算出一个句子的语法解析树，需要一个神经网络循环的训练这个语法解析树，这种结构的神经网络不仅仅是层次序列，而是基于一些图操作或树结构，在每个不同的数据点可能有不同的图或者树结构。这个计算图的结构在输入数据上重复的出现，从数据点上到数据点上可能不同，这种结构看起来有点复杂，用`TensorFlow`很难实现。在`PyTorch`中使用基本的控制流，它们运行的很好。

![Desktop View](/assets/images/20260523/dynamic_graph_application_03.png){: width="600" height="300" }
_动态图应用（来自cs231n）_

#### **Caffe**

下面介绍下`Caffe`框架。

`Caffe`是一个不同于其他架构的深度学习架构，很多时候你不需要写任何代码就可以训练网络，这叫做预存在二进制文件，只需要修改一些配置，不需要任何代码。

![Desktop View](/assets/images/20260523/caffe_overview.png){: width="400" height="200" }
_Caffe（来自cs231n）_

![Desktop View](/assets/images/20260523/caffe_train_and_finetune.png){: width="400" height="200" }
_Caffe 训练和微调（来自cs231n）_

首先将数据格式转换成`HDF5`或者`LMDB`格式，或者可以将图像文件夹或文本文件夹转换成可以进入`Caffe`的脚本。

![Desktop View](/assets/images/20260523/caffe_step_1_convert_data.png){: width="400" height="200" }
_Caffe 第一步：数据格式转换（来自cs231n）_

你需要的定义计算图的结构，而不需要写代码，只需要做的是修改一个叫prototxt的文件，设置计算图的结构，这里是我们从HDF5中读取的结构。我们运行一些内积运算，计算一些loss，这个图的结构在这个文件中设置。

![Desktop View](/assets/images/20260523/caffe_step_2_define_network.png){: width="600" height="300" }
_Caffe 第二步：定义计算图的结构（来自cs231n）_

这些文件的缺点之一是，当网络非常大时，这种设置非常不友好，例如`152`层`ResNet`网络被预训练在`Caffe`中，它的`prototxt`文件有`7000`行，大家基本上不自己写，有种用`python`脚本来自动生成`prototxt`文件。

![Desktop View](/assets/images/20260523/caffe_step_2_define_network_02.png){: width="600" height="300" }
_Caffe 第二步：注意事项（来自cs231n）_

设置一些优化器对象，除了这些你定义一些求解程序，在其它的`prototxt`，这里设置学习率、你的优化算法。

![Desktop View](/assets/images/20260523/caffe_step_3_define_solver.png){: width="600" height="300" }
_Caffe 第三步：设置一些优化器对象（来自cs231n）_

一旦完成所有设置，你可以运行`Caffe`二进制文件利用`train`命令神奇的运行起来。

![Desktop View](/assets/images/20260523/caffe_step_4_train.png){: width="400" height="200" }
_Caffe 第四步：开始训练（来自cs231n）_

`Caffe`有个模型库，放了很多有用的预训练模型。

![Desktop View](/assets/images/20260523/caffe_model_zoo.png){: width="600" height="300" }
_Caffe model zoo（来自cs231n）_

`Caffe`有一个`Python`接口，但是没有很好的说明文档，你需要通过源代码来看。

![Desktop View](/assets/images/20260523/caffe_python_interface.png){: width="400" height="200" }
_Caffe Python接口（来自cs231n）_

对`Caffe`的印象是，具有很好的前向传播模型，很适合产品化，但是它不依赖`python`，在最近的研究中被应用的少了。我还认为它在产品应用方面非常好用。

![Desktop View](/assets/images/20260523/caffe_pros_cons.png){: width="400" height="200" }
_Caffe Pros/Cons（来自cs231n）_

#### **Caffe2**

`Caffe2`是`Caffe`的升级版，它使用静态图，就像`TensorFlow`。和`Caffe`一样，内核是`C++`，有`Python`接口。不同之处，你不再需要写`Python`脚本来生产`prototxt`文件，可以在`Python`中定义自己的计算图结构，就像`TensorFlow`似的`API`，你也可以把它们转化成`prototxt`文件。一旦模型被训练好，就会发现图的好处，你不需要原始的训练代码而发布一个训练模型。

![Desktop View](/assets/images/20260523/caffe2_overview.png){: width="400" height="200" }
_Caffe2（来自cs231n）_

有趣的是，`Google`有一个主用的架构`TensorFlow`，`Facebook`有两个，它们存在有不同的道理，`Google`尝试建立一个网络架构，适用于所有的深度学习场景，所有的效果集合在一个架构上也很好，这意味着，你只需要学习一种架构就可以在所有的场景上应用，包括分布式系统、产品部署、手机端，研究所有的应用场景只需要学习一种架构。而`Facebook`采用不同的策略，`PyTorch`更专业，针对科学研究。在将想法写成研究性代码和更快的迭代实现时，`PyTorch`非常容易实现。如果产品化时，比如手机端，`PyTorch`支持并不太好，而`Caffe2`在这种产品化情况下表现的很好。我的建议是更倾向于针对所有问题的架构。

![Desktop View](/assets/images/20260523/caffe2_suit_production.png){: width="400" height="200" }
_Caffe2在这种产品化情况下表现的很好（来自cs231n）_

`TensorFlow`是一个稳妥的选择，因为它适用所有的环境，适用于所有场景。然而如果你要用动态图，可能需要一个高级的封装。

#### **选型建议**

![Desktop View](/assets/images/20260523/deep_learning_framework_advice.png){: width="400" height="200" }
_Caffe2在这种产品化情况下表现的很好（来自cs231n）_

`PyTorch`非常适合科研，如果只是写研究性代码，`PyTorch`是很好的选择，但它太新了，支持的社区很少，网上的代码少一些。

如果想发布产品，可以选择`Caffe2`和`TensorFlow`，如果是手机端的应用，可以选择`Caffe2`和`TensorFlow`，它们都有内嵌的支持。

没有全局最优的选择，需要根据想要做的事，来决定采用什么架构，决定什么应用。
