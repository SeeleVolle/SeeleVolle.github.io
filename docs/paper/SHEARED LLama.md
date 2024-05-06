# SHEARED LLAMA

Pruning基础知识：

+ Structured vs Unstructured: 非结构化剪枝通常以mask的方式实现，通常针对单个参数而不改变实际参数量，对模型性能损伤少。结构化剪枝是**物理的**移除基于特定规则的网络中的子结构，从而降低网络参数以及推理成本
+ What is pruned: 剪枝实际上是对网络中的模型参数进行mask或者物理pruning，从而降低通道数/权重矩阵规模
+ Stage: 剪枝分为两部分，第一部分是对模型进行修剪来减少参数，第二部分是根据建模目标对模型进行预训练来达到指定性能

正则化：

+ 范数：L0范数指向量中非0元素个数，L1范数指向量中各个元素绝对值之和，L2范数指向量各元素的平方和求平方根
+ 参数稀疏的好处：能实现特征的自动选择，它会学习地去除掉$xi$中大部分和最终输出$yi$没有关系的特征(将特征对应权重置为0)，从而不会干扰对$yi$的正确预测
+ 正则化也就是在训练过程中给Loss加上对应的范数，即$L(W)=\frac{1}{N}\sum L_i+\lambda R(W)$

Relaxation操作：松弛操作，字面意思就是将某个约束条件减弱。以Dijstra算法更新最短路径为例，当$dist(v)\leq dist(u)+w(u,v)$约束条件不满足时，对$dist(v)$进行更新，那么原来的约束条件就已经被满足了，相当于减弱了，因为此时这个约束条件限制不了$dist(v)$了。

提出论点：对现有的LLM进行structured pruning比构建更小规模的LLM更加有效。提出LLM-shearing算法，来解决两个问题

> 如何确定final pruned model是性能强大且推理高效的
>
> 如何继续对pruned model继续进行预训练来达到期望的性能

包含两部分：

1. A novel pruning algorithm：用于将原模型修建到specified target architecture，这个目标架构是由已存在的预训练模型的配置来确定的。修建过程中，能够保持性能并遵循给定的约束条件(非结构化剪枝)
2. 动态batch loading算法：用于在对pruned model进行pre-train的时候，根据每个domain loss减少的速率，按比例加载训练数据，从而有效利用数据并加速整体性能改进。

**Targeted Structured Pruning**

假设source model为$M_s$，有$L_s$个Layer，每一个layer的MHA有$H_s$heads，FFN的intermidiate dimension是$m_S$, 模型的hidden state dimension是$d_S$

针对不同粒度(从global 到 local)有四个mask: $z^{layer}\in R^{L_S}, \ z^{hidden}\in R^{d_S},\ z^{head}\in R^{H_S}(\times L_S), z^{int}\in R^{m_S}(\times L_S)$

<img src="../assets/image-20240426150422929.png" alt="image-20240428161617757" style="zoom: 67%;" />

将pruning描述为限制条件下的优化问题：通过学习pruning masks来找到满足先前指定架构的subnetwork，同时最大化性能。

对mask用Hard concrete distribution进行参数化：

+ Hard concrete distribution: 分布在$[0,1]$上，但概率集中在0或1上，从而实现离散的pruning或retaining。

<img src="../assets/image-20240428161617757.png" alt="image-20240428161617757" style="zoom: 67%;" />

+ mask具体应用在各层的output，transformer中mlp和attention的output都有mask

使用一对拉格朗日乘子直接对修剪后的模型形状施加约束

例如$H_\gamma$限制：$L^{head}(\lambda,\phi,z)=\lambda^{head}\cdot(\sum z^{head}-H_{\gamma})+\phi^{head}\cdot (\sum z^{head}-H_\gamma)^2$，其中$H_\gamma$表示targeted number of heads，对于其他的target也是类似

最终优化objective function：$L_{prune}(\theta,z,\lambda,\phi)=L(\theta,z)+\sum_{j=1}^{L_S}L_j^{head}+\sum_{j=1}^{L_S}L_j^{int}+L^{layer}+L^{hidden}$，其中$L(\theta,z)$是使用masked model weight计算后的loss。需要最小化模型权重$\theta$和修剪掩码$z$，并找到$\lambda$和$\phi$满足对应的限制函数。

由于mask会快速收敛，并且pruning耗时很长，所以对pruning budget进行限制。pruning之后，保留每个子结构中**the highest-scoring components associated with the mask variables**，然后在继续进行pre-training.

**Dynamic Batch Loading**

+ 实时根据loss动态调整domain proportion.

假设一个预训练集由$D_1,...D_n$组成，对于每个domain验证集是$D_i^{val}$，每个step，$D_i$有一个proportion $\omega_t[i]$，一个reference loss $l_{ref}D_i$。

具体过程如下：简单来说，首先在每个Domain上得到一个参考loss，然后每训练m步，就针对每个Domain计算得到一个loss，然后用这个loss来对proportion进行更新。根据对应的proportion在每个Domain上取样，然后根据之前的Objective function对模型参数，mask，以及一组拉格朗日参数进行更新。

Reference的选择：Source reference和Scaling reference

+ Source model’s domain validation loss
+ Fitted scaling function（用一个现有模型的loss来拟合函数）<img src="../assets/image-20240428152658367.png" alt="image-20240428152658367" style="zoom:67%;" />

+ Reference loss of Scaling laws:

$L(N,D)=E+\dfrac{A}{N^\alpha}+\dfrac{B}{D^\beta}$，其中$N$是模型大小，$D$是数据集大小，$E$是理想生成过程中的loss，其他都是缩放因子。对于一类的Model，每个Domain的scaling factors都是不同的，需要单独估计，并且需要多个不同大小的模型。具体来说，使用多个模型的checkpoints，在每个Domain的验证集上进行evaluate来拟合scaling factors。