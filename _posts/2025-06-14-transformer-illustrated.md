---
title: The Illustrated Transformer（翻译版）
date: 2025-06-14 20:00:00 +0800
categories: [AI, NLP]
tags: [AI, NLP, Transformer]
music-id: 1908182683
---

>本文译自  [**The Illustrated Transformer**](https://jalammar.github.io/illustrated-transformer/)
{: .prompt-tip }

## **Overview**
`Attention is All You Need`论文原文中介绍，`Transformer`模型，完全摒弃了`RNN`和`CNN`的先验假设，它是一个只使用了`Attention`机制的`Seq2Seq`模型，通过`Attention`机制连接两部分——`Encoder`和`Decoder`完成`Seq2Seq`任务。

![Desktop View](/assets/img/20250614/transformer_01_overview.png){: width="500" height="300" }
_Transformer seq2seq简图_

### **Transformer的特点**
1. 无先验假设，相比于`CNN`和`RNN`，优点是能更快地学到无论长时还是短时的关联性。
- `CNN`的先验假设，是局部关联，`Transformer`对相对位置不敏感，可以并行计算；
- `RNN`的先验假设，是有序建模，`Transformer`需要位置编码来反映位置变化对于特征的影响，对绝对位置不敏感；
- 先验假设越多，人为注入经验，模型更容易学，需要的数据量就越低；
- 数据量的要求与先验假设的程度成反比。

2. 核心计算在自注意力机制，序列越长，计算复杂度是平方增长的。
- `RNN`中，有序建模，递归地去算，每次计算计算量是固定的；
- 有很多工作，是去降低复杂度，要降低就得注入先验假设，比如算注意力不是每个位置都算，局部关联或者单调性；
- 要用好`Transformer`模型，需要根据不同的任务，去注入一些任务相关的先验假设，比如在注意力机制、`loss`、模型结构上，根据这些先验假设，做些优化。

细分下上图，论文原文中`Encoder`部分由`6`个编码器堆叠组成，`Decoder`也是，这里的数字`6`并没啥特殊的含义，也可以尝试用别的数字，比如`2`。

![Desktop View](/assets/img/20250614/transformer_02_encoder_6_decoder.png){: width="500" height="300" }
_原论文中Transformer包含6个Encoder、6个Decoder_

简单看下`Encoder`和`Decoder`的组成，
- 编码器的输入，先经过一个自注意力层，这一层能帮助编码器在编码特定单词时查看输入句子中的其他单词。自注意力层的输出被喂给`FFN`，完全相同的`FFN`可以被独立地应用于每个位置。
- 解码器也包含这些层，只不过相比编码器，中间多一个注意力层，这一层帮助解码器专注于输入句子的相关部分。

![Desktop View](/assets/img/20250614/transformer_03_encoder_component.png){: width="500" height="300" }
_Transformer Encoder和Decoder组成简图_

假设把前面的超参数字`6`改成`2`，就是一个由`2`个堆叠的编码器和解码器组成的`Transformer`，如下，

![Desktop View](/assets/img/20250614/transformer_04_demo_2_layers.png){: width="500" height="300" }
_Transformer 2层Encoder+Decoder架构简图_

再参照论文里边的`Transformer`模型架构图，如下，

![Desktop View](/assets/img/20250614/transformer_05_arch.png){: width="500" height="300" }
_Transformer 论文原架构图_

## **Encoder**
现在我们已经看到了模型的主要组成部分，让我们开始看看各种张量是如何在这些组成部分之间流动的，从而将训练模型的输入转换为输出的。

### **Input Embedding**
与一般的`NLP`流程一样，先用`Embedding`算法把每个输入的单词转换为向量。

![Desktop View](/assets/img/20250614/transformer_06_input_embedding.png){: width="500" height="300" }
_Transformer Input Embedding_

论文里的`embedding`维度，每个单词都`embedding`到大小为`512`的向量，暂且用这些简单的方框来表示这些向量。

注：`embedding`仅发生在最底部的`Encoder`中。所有编码器的共同抽象是，它们接收一个大小为`512`的向量列表——在底部编码器中是单词的`embedding`，但在其他编码器中的输入，是直接位于下方的编码器的输出。这个列表的大小是可以设置的超参数——一般是我们训练数据集中最长句子的长度。

### **Position Embedding**
这是一个描述`input`序列中单词顺序的方法。

`Transformer`在每个输入`embedding`中加一个位置向量，帮助它确定每个单词的位置，或序列中不同单词之间的距离。

直观上看，一旦嵌入向量被投影到`Q`/`K`/`V`向量中，并在点积注意力期间，将这些值添加到`embedding`向量中，可以在`embedding`向量之间提供有意义的距离。

![Desktop View](/assets/img/20250614/transformer_07_position_embedding.png){: width="500" height="300" }
_Transformer Position Embedding_

在我们的输入序列中`embedding`单词和位置后，每个单词都会流过编码器的两层，分别是`Self-Attention`、`FFN`。

![Desktop View](/assets/img/20250614/transformer_08_embedding_to_sa_ffn.png){: width="500" height="300" }
_Transformer embedding流过encoder的两层_

在这里，有一个`Transformer`的一个关键属性——每个位置的单词在编码器中都沿着自己的路径流动。在自注意力层中，这些路径之间存在依赖关系。但`FFN`层不具有这些依赖性，因此各种路径可以在流经`FFN`层时并行执行。

<font color="red">接下来，我们把例子切换到一个较短的句子，看看编码器的每个子层中发生了什么。</font>

### **Now We’re Encoding**
正如我们已经提到的，编码器接收向量列表作为`input`，它将这些向量传递到自注意力层，然后传递到`FFN`来处理这个列表，然后将输出向上发送到下一个编码器。

![Desktop View](/assets/img/20250614/transformer_09_encoding.png){: width="500" height="300" }
_Transformer Encoding_

### **Self-Attention**
`Self-Attention`怎么理解？

假设以下是我们要翻译的句子，

`The animal didn't cross the street because it was too tired`

这个句子中的`it`指的是什么？它指的是街道还是动物？这对人来说是一个简单的问题，但对算法来说却不那么简单。

当模型处理单词`it`时，自注意力使其能够将`it`与“动物”联系起来。

当模型处理每个单词（输入序列中的每个位置）时，自注意力允许它查看输入序列中其他位置的线索，以帮助更好地编码这个单词。

如果熟悉`RNN`，想想维护隐藏状态是如何让`RNN`将它处理过的先前单词/向量的表示，与它正在处理的当前单词/向量结合起来的。自注意力层是`Transformer`用来将其他相关单词的“理解”加工到我们当前正在处理的单词编码的方法。

![Desktop View](/assets/img/20250614/transformer_10_understand_self_attention.png){: width="500" height="300" }
_Self-Attention 直观理解_

当我们在编码器`5`（`Encoders`堆栈中的顶部编码器）中对单词`it`进行编码时，部分注意力机制集中在`the Animal`上，并将其表示的一部分加工成`it`的编码。

<font color="red">接下来，我们把示例切换到一个较短的句子，让我们先看看，怎么使用向量计算自注意力Self-Attention？</font>

**第一步**，从编码器的每个`input`向量（每个单词的`embedding`）中创建三个向量，即对每个单词都创建一个查询向量`q`、一个关键字向量`k`和一个值向量`v`。这些向量是通过将`embedding`乘以在训练过程中训练的三个参数矩阵得到的。

注意，这些新向量的维度小于`embedding`向量，论文中它们的维度是`64`，而`embedding`和编码器输入/输出向量的维度为`512`。它们不必更小，这是一种架构选择，可以使多头注意力的计算（大多）保持不变。

![Desktop View](/assets/img/20250614/transformer_11_matrix_calc_step_01.png){: width="500" height="300" }
_Self-Attention 向量计算第一步，得到q、k、v_

乘以`WQ`权重矩阵得到`q1`，即与该单词相关的查询向量。我们最终为`input`句子中的每个单词分别创建了一个`query`、一个`key`和一个`value`投影。

<font color="red">什么是Q、K、V向量？</font>

它们是用于计算和思考注意力的抽象概念。继续往下看注意力的计算方法，就可以知道所有你要知道的。

第二步，计算分数`Score`。假设我们正在计算这个例子中第一个单词`think`的自注意力。需要根据这个单词对`input`中的每个单词进行评分。分数决定了当我们在某个位置对单词进行编码时，对`input`其他部分的关注程度。

分数是通过`Q`向量与我们正在评分的相应单词的`K`向量的点积来计算的。因此，如果我们正在处理位置`1`中单词的自注意力，第一个分数将是`q1`和`k1`的点积。第二个分数是`q1`和`k2`的点积。

![Desktop View](/assets/img/20250614/transformer_12_matrix_calc_step_02.png){: width="500" height="300" }
_Self-Attention 向量计算第二步，计算分数score_

第三步和第四步，将分数除以`8`（这样做是为了更稳定的梯度，论文中使用的是`K`向量维数`64`的平方根，这里也能用其他可能的值，但这里用的是默认值`8`），然后通过`softmax`函数处理最终结果（`Softmax`对分数进行归一化处理，使其全部为正，加起来为`1`）。

![Desktop View](/assets/img/20250614/transformer_13_matrix_calc_step_03.png){: width="500" height="300" }
_Self-Attention 向量计算第三、四步，softmax_

这个`softmax`分数决定了每个单词在这个位置的表达量，这个位置的单词拥有最高的`softmax`分数，但关注与当前单词相关的另一个单词才是有用的。

第五步，将每个`V`向量乘以`softmax`分数（准备将它们相加）。直观上看，是想保持我们想要关注的单词的值不变，并遮掩不相关的单词（例如，通过将它们乘以`0.001`这样的小数字）。

第六步，对加权的`V`向量求和，在这个位置（第一个单词）产生自注意力层的输出。

![Desktop View](/assets/img/20250614/transformer_14_matrix_calc_step_05.png){: width="500" height="300" }
_Self-Attention 向量计算第五、六步，产生自注意力层的输出_

自注意力层的计算，到此结束，得到的向量发送到`FFN`。在实际实现中，为了更快的处理，这种计算是以矩阵形式完成的。

**继续看看矩阵计算如何实际实现？**

第一步，计算`Q`、`K`和`V`矩阵，将每个单词的`embedding`组合打包到矩阵X中，并将其乘以我们训练的权重矩阵（`WQ`、`WK`、`WV`）。

![Desktop View](/assets/img/20250614/transformer_15_matrix_calc_do_step_01.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，第一步_

`X`矩阵中的每一行对应于输入句子中的一个单词，上图用小框区别了`Embedding`向量（`512`，或图中的`4`个框）和`q`/`k`/`v`向量（图中的`64`，或`3`个框）的大小差异

因为我们处理的是矩阵，我们可以将第二步到第六步压缩到一个公式中，以计算自注意里层的输出。

![Desktop View](/assets/img/20250614/transformer_16_matrix_calc_do_step_02_06.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，第二步到第六步_

### **Multi-head Self-attention**
论文里，通过添加**多头注意力**机制，进一步细化自注意力层。从两个方面提高了注意力层的性能，
- 它扩展了模型专注于不同位置的能力。在上面的例子中，`z1`包含了其他编码的一小部分，但它可能被实际的单词本身所主导。如果我们翻译一个句子，比如`The animal didn’t cross the street because it was too tired`，知道`it`指的是哪个词会很有用。
- 它为注意力层提供了多个“表示子空间”。使用多头注意力，我们不仅有一组，而是有多组`Q`/`K`/`V`权重矩阵（`Transformer`使用八个注意力头，因此每个编码器/解码器最终都有八组）。这些集合中的每一个权重矩阵都是随机初始化的。然后，在训练之后，使用每个集合将输入`Embedding`（或来自较低编码器/解码器的向量）投影到不同的表示子空间中。

![Desktop View](/assets/img/20250614/transformer_17_matrix_calc_multi_head.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，多头自注意力_

通过多头注意力，我们为每个头维护单独的`Q`/`K`/`V`权重矩阵，从而得到不同的`Q`/`K`/`V`矩阵。与之前一样，我们将`X`乘以`WQ`/`WK`/`WV`矩阵，得到`Q`/`K`/`V`矩阵。

如果我们进行上面说的相同的自注意力计算，只需使用不同的权重矩阵进行八次不同的计算，我们最终会得到八个不同的`Z`矩阵。

![Desktop View](/assets/img/20250614/transformer_18_matrix_calc_multi_head_z.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，多头自注意力结果得到Z矩阵_

但`FFN`层不需要八个矩阵，它只需要一个矩阵（每个单词一个向量），所以我们需要一种方法将这八个压缩成一个矩阵。

我们`concat`这些矩阵，然后将它们乘以一个额外的权重矩阵`WO`。

![Desktop View](/assets/img/20250614/transformer_19_matrix_calc_multi_head_wo.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，Z矩阵经过WO，输出结果_

所有步骤，合在一起，放在一张图里边，如下，

![Desktop View](/assets/img/20250614/transformer_20_matrix_calc_summary.png){: width="500" height="300" }
_Self-Attention 向量计算实际实现，汇总_

现在我们已经谈到了注意力头，让我们重新审视之前的例子，看看在我们的例句中编码单词`it`时，不同的注意力头都集中在哪里：

![Desktop View](/assets/img/20250614/transformer_21_understand_self_attention_layer_5.png){: width="500" height="300" }
_Self-Attention 理解_

当我们对单词`it`进行编码时，一个注意力主要集中在`the animal`上，而另一个注意力则集中在`tired`上——从某种意义上说，模型对单词`it`的表示在`animal`和`tired`的一些表示中起到了推波助澜的作用。

如果我们把所有的注意力都放在图片上，事情可能会更难解释，

![Desktop View](/assets/img/20250614/transformer_22_understand_self_attention_layer_5.png){: width="500" height="300" }
_Self-Attention 理解_

### **LayerNorm & Residual**
每个编码器中的每个子层（`Self-Attention`，`FFN`）都有一个围绕它的残差连接，然后是层归一化步骤。

![Desktop View](/assets/img/20250614/transformer_23_residual.png){: width="500" height="300" }
_Self-Attention 层归一化 & 残差_

可视化向量和与自注意力相关的`layer-norm`操作，如下，

![Desktop View](/assets/img/20250614/transformer_24_residual_visual.png){: width="500" height="300" }
_向量可视化_

### **Feed-forward Neural Network**
`FFN`层的作用是，通过放大再缩小的方式，在注意力层输出的特征里边寻找有用的特征，多样化，提升特征表征能力，论文里边的维度是`512`（`input`）-`2048`（中间）-`512`（`output`）。

## **Decoder**
<font color="red">编码器和解码器是如何协同工作的？</font>

编码器首先处理`input`序列，顶部编码器的输出被转换为一组注意力向量`K`和`V`。这些向量将由每个解码器在其`Encoder-Decoder Attention`层中使用，帮助解码器专注于输入序列中的适当位置。

![Desktop View](/assets/img/20250614/transformer_decoding_1.gif){: width="600" height="400" }
_Decoder 一组处理过程-动态图_

完成编码阶段后，开始解码阶段。解码阶段的每一步都从输出序列中输出一个元素。

重复这个过程，直到达到一个特殊符号，表示`Transformer`解码器已完成其输出。每一步的输出在下一刻被喂给底部解码器，解码器像编码器一样放大解码结果，就像我们对编码器输入所做的那样，将位置编码`Embedding`，并添加到这些解码器输入中，以指示每个单词的位置。

![Desktop View](/assets/img/20250614/transformer_decoding_2.gif){: width="600" height="400" }
_Decoder 全部处理过程-动态图_

解码器中的自注意力层的工作方式与编码器中的略有不同。

在解码器中，自注意力层只允许关注输出序列中的较早位置。通过在自注意力计算中的`softmax`步骤之前屏蔽未来的位置（将其设置为`-inf`）来实现的。这一步骤也叫<font color="red">Casual Multi-head Self-attention</font>。

`Encoder-Decoder Attention`层的工作原理，除了它从下面的层创建`Q`矩阵，并从编码器堆栈的输出中获取`K`和`V`矩阵，与多头自注意力一样。这一步骤也叫<font color="red">Memory-base Multi-head Cross-attention</font>，或者`Intra-attention`。

## **The Final Linear and Softmax Layer**
`Decoder`输出一个`floats`向量，如何把它变成一个词？这就是`Linear`层+`Softmax`层的工作。

`Linear`层是一个简单的全连接神经网络，它将`Decoder`产生的向量投影到一个更大的向量中，称为`logits`向量。

假设模型知道`10000`个独特的英语单词（模型的“输出词汇表”），这些单词是从它的训练数据集中学习到的。这将使`logits`向量的宽度达到`10000`个单元格，每个单元格对应一个唯一单词的分数。这就是我们如何解释线性层后面的模型输出。

然后，`softmax`层将这些分数转换为概率（全部为正，加起来为`1.0`）。选择概率最高的单元格，并产生与之相关的单词作为此时刻的输出。

![Desktop View](/assets/img/20250614/transformer_25_linear_softmax.png){: width="500" height="300" }
_Linear + Softmax_

这张图从底部往上看，以`Decoder`的输出向量为起点，然后将其转换为输出单词。

## **Training**
现在我们已经通过一个训练好的`Transformer`完成了整个前向传递过程，看看训练模型的过程会对理解很有帮助。

在训练过程中，未经训练的模型将经历完全相同的前向传递。但因为是在标记的训练数据集上训练它的，所以可以将其输出与实际的正确输出进行比较。

为了直观地展示这一点，假设输出词汇表只包含六个单词（`a`、`am`、`i`、`谢谢`、`学生`和`<eos>`（句末的缩写）），模型的输出词汇表是在我们开始训练之前的预处理阶段创建的。

![Desktop View](/assets/img/20250614/transformer_26_train_vocab.png){: width="500" height="300" }
_词汇表_

一旦我们定义了输出词汇表，就可以使用相同宽度的向量来表示词汇表中的每个单词，这也被称为`one-hot`编码。例如，我们可以使用以下向量表示单词`am`，

![Desktop View](/assets/img/20250614/transformer_26_train_vocab_onehot.png){: width="500" height="300" }
_词汇表 one-hot_

示例，输出词汇表的一个`one-hot`编码

之后，讨论一下模型的损失函数——我们在训练阶段优化的指标，以得到一个训练过的、希望非常准确的模型。

## **The Loss Function**
假设正在训练模型，这是在训练阶段的第一步，正在用一个简单的例子来训练它——将`merci`翻译成`thanks`。

这意味着，希望输出是一个表示`thanks`这个词的概率分布，但由于该模型尚未经过训练，因此目前不太可能发生这种情况。

![Desktop View](/assets/img/20250614/transformer_27_loss_expect.png){: width="500" height="300" }
_损失和真值期望_

由于模型的参数（权重）都是随机初始化的，（未经训练的）模型为每个单元格/单词生成具有任意值的概率分布。我们可以将其与实际输出进行比较，然后使用反向传播调整模型的所有权重，使输出更接近所需的输出。

如何比较两种概率分布？我们只是从另一个中减去一个。有关更多细节，请查看[cross-entropy](https://colah.github.io/posts/2015-09-Visual-Information/) 和 [Kullback–Leiblerdivergence](https://www.countbayesie.com/blog/2017/5/9/kullback-leibler-divergence-explained)。

注意，这是一个非常简单的例子。实际上，我们会用比一个单词更长的句子。

例如，输入：`je suisétudiant`，预期输出：`i am a student`。

这意味着，我们希望模型能够连续输出概率分布。

其中：

每个概率分布都由一个宽度为`vocab_size`的向量表示（在我们的简单示例中`vocab_size`为`6`，但现实中可能是像`30000`或`50000`这样的数字）。

第一个概率分布在与单词`i`关联的单元格处具有最高概率。

第二概率分布在与单词`am`关联的单元格处具有最高概率。

以此类推，直到第五个输出分布指示`<end of sentence>`符号，该符号在`10000`个元素词汇表中也有一个与之关联的单元格。

![Desktop View](/assets/img/20250614/transformer_28_loss_target_model_outputs.png){: width="500" height="300" }
_以上是，在一个样本句子的训练示例中，训练模型的目标概率分布_

在足够大的数据集上，训练模型足够长的时间后，希望生成的概率分布看起来像这样，

![Desktop View](/assets/img/20250614/transformer_29_loss_trained_model_outputs.png){: width="500" height="300" }
_模型在实际被训练后的概率分布_

希望经过训练，模型能够输出我们期望的正确翻译。当然，这并不能真正表明这个短语是否是训练数据集的一部分（参见：[cross validation](https://www.youtube.com/watch?v=TIgfjmp-4BA)）。注意，每个位置都有一点概率，即使它不太可能是该时刻的输出——这是`softmax`的一个非常有用的属性，有助于训练过程。

现在，因为模型一次产生一个输出，可以假设模型正在从概率分布中选择概率最高的单词，并丢弃其余的单词。这是一种方法（称为贪婪解码，`greedy decoding`）。

另一种方法是，找到前两个单词（例如`I`和`a`），然后在下一步中运行模型两次：一次假设第一个输出位置是单词`I`，另一次假设第二个输出位置为单词`a`，并且保留位置`1`和`2`产生较少错误的版本，对位置`2`和`3`等重复此操作。这种方法被称为`beam search`。

在例子中，`beam_size`为`2`（这意味着，在任何时候，两个部分假设（未完成的翻译）都保存在内存中），`top_beams`也为`2`（意味着我们将返回两个翻译）。这两个参数都是你可以去试验的超参数。

**END**
