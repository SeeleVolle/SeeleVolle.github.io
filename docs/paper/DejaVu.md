# De jaVu:Conditional Regenerative Learning to Enhance Dense Prediction: 

来自法语，表达既视感的意思，这不由得让我想起了石头门(x

Sparsity之前的一些结果：

+ Iterative pruning and lottery ticket hypothesis: Lee et al., 2018,  Frankle & Carbin, 2018
+ Task-dependent pruning: Michel et al., 2019, Bansal et al., 2022
+ zero-shot pruning: SparseGPT

> It is hard to achieve wall-clock time speed-up with unstructured sparsity

The Hardware Lottery: 由Hooker提出的一个观点，指的是一个研究方案的胜出往往并不是它全方面优于其他方案，而是它们更适合当时的软硬件环境。

Sparsity ratio：指的是权重矩阵中0元素的比例，而pruning就是将权重矩阵中的部分值舍去（变为0），或者更形象的说是将模型中的一些参数去掉或者某些参数与参数之间的连接去掉

+ structured pruning: 指的是将权重矩阵进行整行的消除，相当于相当于减少了全连接层的一个neuron或者一个卷积层的channel
+ unstructured pruning: 使用的是mask，而不是直接进行消除。
+ 为什么pruning能够减小内存使用：通过将剪枝后的权重矩阵$W_p$以特定的方式存储并结合特定优化后的运算才能减少内存使用并实现推理加速

Lottery ticket hypothesis：All you need is the winning tickets..... 在一个神经网络中，始终存在一个较小的子网络，它们重新开始训练，并且至少与较大的子网络一样的学习，能够同时达到类似的测试精度。但是这些子网络需要合适的初始化才能进行有效的训练

Ideal sparsity for LLMs: 

1. 不需要模型重新训练
2. 保持了模型的质量和上下文学习能力
3. 在现代硬件上有wall-clock time加速(wall-clock time指的是CPU Time+ off-CPU Time，主要包含了CPU的执行时间和I/O时间)

Contextual sparsity: 一种小型的，依赖输入的注意力头和MLP参数集的sparsity，对于某个输入，它可以和完整的模型产生近似的输出。

+ Hidden Markov Models: 缩写为HMM，一种统计学模型，简单来说就是未来状态不仅取决于当前状态，还取决于隐藏状态。即引申出了观测变量和隐变量的两个概念。

> 用数学模型来表示，有一个结果集合$X={x1, x2}$，一个状态转换矩阵，一个$Emission=(emission_k(b))$矩阵(状态k出现在b结果的可能性)，一个hidden path: $\pi=\pi_1 \pi_2...\pi_n$，我们关注的是$Pr(x,\pi)$即HMM中状态序列$\pi$对应结果序列为$x$的概率，或者是$Pr(x,\pi)$即在状态序列$\pi$下的条件概率，通常我们希望找到一个hidden path $\pi$使得$Pr(x,\pi)=Pr(x|\pi)*Pr(\pi)$最大，此时结果序列是已知的即观测变量，而hidden path则是隐变量

+ Viterbi algorithm：一种解决HMM问题的动态规划算法(存储最短+提前剪枝)，具体没有细看



作者关于Contextual sparsity解决了三个问题：Existence, Prediction, Efficiency

+ Existence: 为了达到基本相同的输出，Contextual sparsity平均是 85% structured sparsity，在保持准确性的同时，每个特定输入可能会导致参数减少 7 倍
+ Prediction: 
  + Contextual sparsity取决于两方面，依赖于Individual input tokens(non-contextual dynamic sparsity)和它们之间的interaction(contextual dynamic sparsity)即足够的上下文信息。
  + contextual dynamic sparsity可以由Head/MLP的参数和上一层output之间的相似性来预测，output中带有the immediate contextual mixture of token embeddings。
+ Efficiency:
  + 将基于相似性的预测问题制定为一个NNS问题(Nearest Neighbor Search)
  + Token Embeddings在各层之间变化缓慢(由于残差连接)，即连续几层的输入十分相似，所以可以设计一个asynchronous lookahead predictor.

> 对于某一层的特定输入，可以预测下一层相关的subset of attention heads or MLP parameters，这样就只用Load相关参数来计算
>
> 异步预测器可以在理论上保证sparsity prediction的准确性
>
> 整合hardware-aware的稀疏矩阵乘法实现

Typical LLM procedure：

+ prompt：根据输入序列在每一层Tf layer产生KV cache
+ token generation：使用KV cache来一步一步生成token同时更新KV cache
  + 在token generation中，占比时间主要由MLP、attention和GPU communication

**Sparsified MLP**：

$MLP_{S_M(y)}=\sigma(yW^1_{S_M})(W^2_{S_M})^T$，其中$\sigma$是activation function, $W_1$和$W_2$是两个线性层，$y$是Input，下标$S_M\subseteq[4d]$指的是Neuron的子集即所有列的某一个子集

**Sparsified Attention**：

$MHA_{S_A}(y)=\sum_{i\in S_A}H_i(y)W_i^O$，其中$H_i(y)$是$1 \times d_h$的矩阵，$W_i^O$是$d_h\times d$的矩阵。各部分的意思如下

+ $W_i^O$是output在$i$-th attention head上的投影，$W_i^K$, $W_i^Q$, $W_i^V$同理是$d\times d_h$的
+ $S_A$是heads的一个子集， $y$是MHA在current generation step的input，一共有$h$个head
+ $H_i(y)=D_i(y)^{-1}exp(yW_i^Q(W_i^K)^TX^T)XW_i^V$, 其中$D_i(y)=exp(yW_i^Q(W_i^K)^TX^T)1_n$，$H_i(y)$是$1\times d_h$的
+ $X\subseteq R^{n\times d}$是之前所有token的embeddings，包含prompts和先前生成的token

问题转化为了找$S_M$和$S_A$使得sparse approximation和full computation的结果误差在可接受范围内

**LLM Contextually Sparse**：三大发现

+ LLM中确实存在Contextually Sparse

  + 对于MLP block来说，由于其中的activation function，所以可以直观的发现其中的contextual sparsity，即和输入x点积最大的neuron

  + 对于attention block，对于不同Input有不同的contextual sparsity
    + bypass uniform heads(small output norms) and selective “heavy hitter" heads

+ self-attention是一个聚类(clustering)算法：Mean-shift clustering

  + Mean-shift clustering：一种聚类算法

   <img src="../assets/image-20240423184001562.png" alt="image-20240423184001562" style="zoom:50%;" />

  > 证明过程，构造了一个类似mean shift算法中的计算平均位移向量的函数$m_i(y)=\frac{\sum_jK_i(x_j,y)x_j}{\sum_jK_i(xj,y)}$，其中定义$K_i(x_j,y)=exp(yW_i^Q(W_i^K)^Tx_j)$来评估$x_j$和$y$的similarity。然后我们可以得到$H_i(y)=m_i(y)W_i^V$，当$W_I^V=I$的时候，加上残差连接，$H_i(y)=Norm(y+m_i(y))$，显然有一个$y=\gamma m_i(y)$的不动点。与mean shift中$y_{i}^{t+1}=m(y_i^t)$的不动点$y_i=m(y_i)$十分类似。所以self-attention一直在做类似shift的聚类操作。

+ 在连续的多个layer之间embeddings十分相似，变化很慢。因为残差连接中$X$的正则项远远大于$F(X)$很大，导致$X+F(X)$变化不大

  + 证明了存在$0<\epsilon_1<\epsilon_2<1$，对于残差连接$y=F(x)+x$，当$F(x)$是MLP或者Attention时，均有$\epsilon_1\leq||y-x||_2\leq \epsilon_2$，即L2范数在0-1之间

  

**Deja Vu Framework**

+ Sparsity operator for MLP：所有层的predictor用一组输入来同时训练

转化为NNS问题，目标是寻找与Input会产生high inner-product的神经元，所以将MLP层的Contextual sparsity predication公式化为内积度量下的经典NNS问题。Approximate MaxIP，近似的最大内积搜索问题

Formulation：$c\in(0,1), \gamma\in(0,1)$，在单位球上给定一个n维向量数据集$W^1\subset S^{d-1}$，目标是建立一个数据结构：对于一个满足$max_{\omega\in W^1}<y,\omega>\geq \gamma$的query $y\in S^{d-1}$，可以检索到一个向量$z\in W^1$，满足$<y,z>\geq c\cdot max_{\omega\in W_1}<y,\omega>$。用人话来说就是，通过满足条件$W^1$中的向量和$y$的最大点积大于某个值时，可以在$W^1$中找到和$y$有最大点积的$z$。

可以将$W^1$看作是第一个线性层，$y$是input embedding.

每一个MLP用一个两层的全连接网络来作为近邻搜索方法，从而进行sparsity prediction，然后再进行真正的预测得到子集$S_M$

问题如何训练的

<img src="../assets/image-20240423201716464.png" alt="image-20240423201716464" style="zoom:50%;" />

+ Sparsity operator for Attention：所有层的predictor用一组输入来同时训练

同样转化为NNS问题，因为每个head都针对一个feature在做mean-shifting。由于Transformer具有token-mixing的特性，所以current token embedding就足够了(解决了attention block对之前token的dependency)，通过Input $y$和head parameters的相似性即之前的$K_i(x_j,y)$来做预测。

将每一个head看作是一类，都进行一次类MLP的Training得到SP(一个两层的全连接网络)，然后进行真正的预测得到head的子集$S_A$。

为了解决缺少之前tokens KV cache的问题，利用了Generation是I/O bound的事实。对于预测后的$S_A$和input $y$，直接计算kv存储到KV cache中，并为其他head保存一份$y$的token embedding副本。之后缺少时，直接reload 已经存储的token embeddings并再次计算kv即可。

+ Reducing Overhead with Asynchronous Execution

为了解决sparsity prediction仍然会带来latency的问题，提出了一个asynchronous look-ahead sparse predition method，利用连续layer之间输入相似的特性来进行cross-layer prediction

将不同Transformer Layer中的多个block进行parallel sparse prediction，在下图中在计算当前Tf layer的$y_l$时，可以计算下一层的$S_A$和$S_M$

<img src="../assets/image-20240423211926524.png" alt="image-20240423211926524" style="zoom:50%;" />



+ Hardware-efficient Implementation
  + Kernel fusion
  + Memory coalescing