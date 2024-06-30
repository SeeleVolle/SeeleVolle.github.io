# FastDecode:High-Throughput GPU-Efficient LLM Serving using Heterogeneous Pipelines

简介：将Transformer模型分成R-Part和S-Part两个部分，key insight是应该在CPU上计算near KV-cache, 通过传递activation tensors而不是KV-cache来减少data transmission，完全将KV-cache从GPU memory中移除。

但是这样做会面临一些问题

+ 给CPU额外增添计算任务可能会减缓所有任务的速度，CPU-GPU之间的PCIE带宽较低
+ 两个Part中工作负载的变化模式不同，交替使用CPU-GPU计算时，两者速度差距很大，不能很好的全面利用CPU和GPU
+ 需要对GPU和CPU进行Careful orchestration，需要使用最少的CPU同时能够最大化的使用GPU的计算能力

本文提出了以下工作：

+ 将Transformer模型分为了两个部分执行来提高performance
+ 提出了基于KV-cache的near-memory processing system, 使用out-of chassis CPUs的聚合内存带宽来获得更高的吞吐量
+ 提出了一个sequence-level pipeline schedule来平衡两种工作负载，来更好的使用两种硬件
+ 建立了一个性能模型来位不同的model和需求提供最佳的硬件配置

**R-Part**: 指的是和之前token有关的auto-regressive的计算，具体包括了$QK, softmax, O_i=\sum_{j=1}^{i-1}A_{ij}V_j$的计算，这些计算依赖于自己的KV-cache，基本不受益于batching
**S-Part**: 指的是model剩余的，所有的sequence共享相同参数的计算，主要包括了全连接层的计算，当batch的token数增大时，GPU利用率会提高

由于R-Part主要是attention计算，是memory-bound的，所以应该存在CPU的DRAM中，并由CPU处理，但是CPU的速度相对较慢以及R-S Part之间传输速度的限制都存在问题。

### System Overview

<img src="../assets/image_FastDecode1.png" alt="image-FastDecode1" style="zoom:80%;" />


有两种worker，R-worker和S-worker，S-worker负责处理S-part的任务，模型的所有权重都在S-worker上，并通过model parallelism分布到多GPU上，和一般的token geneartion worker相比较来说batch size更大，并且不承担attention的计算。

$Q_i, K_i, V_i$在S-part产生之后，会被S-worker发送到不同的对应R-worker上，并获取最终的输出$O_i$。

R-worker利用remote CPUs进行计算，主要负责的就是$QK, softmax, O_i$计算, 保存当前序列的KV-cache。由于S-worker不用保存KV-cache, 所以batch size可以大幅增加，但是过度增加也会超过GPU的计算能力从而造成更大的延迟。

两种worker交替生成token(no pipeline)，一个工作则另一个停止。所以可以采用two-stage pipeline的形式，但是由于workload的处理时间不同，所以不可避免的会产生bubble。

<img src="../assets/image_FastDecode2.png" alt="image-FastDecode2" style="zoom:80%;" />

#### Sequence-level Load-Stabilizing Schedule

为了解决上述提到的bubble问题，需要知道S-part的latency跟batch size成正比，R-part的latency跟总序列长度成正比。所以随着生成token数的增加，R-part的latency一直在增加而S-part没有变化。

因此一开始S-part的latency较高时，R-workers idling, 当达到某个point时，就变成了R-part的latency较高，S-workers idling

<img src="../assets/image_FastDecode3.png" alt="image-FastDecode3" style="zoom:80%;" />

如该图所示，相较于固定的batch size，使用较小的micro-batches来周期性的处理(F steps)。这样相较于原始的R-part，同一时刻的生成的token数更少，因此负载更小，并且负载会逐渐增大保持稳定

B为原sequence数目，S为sequence长度，F是step

此时最大total sequence length = $\sum_{k=1}^{S/F}MkF=\frac{B(S+F)}{2}\approx \frac{BS}{2}=\frac{W_{max}}{2}$

除此之外，对于一个请求处理来说，原先需要等待S steps来处理，现在只需要等待F steps.

**load control algorithm**

<img src="../assets/image_FastDecode4.png" alt="image-FastDecode4" style="zoom:80%;" />

在这个算法中有两个函数，ADDMICROBATCH所做的事情是增加一个新的MICROBATCH时，维护current micro-batches的总batch size, ending step index array以及workload array。

GETEARILIESTSTEP代表的是最早可以启动负载的索引，micro-batch处理完成时刻total sequence length最大，即workload此时达到最大。所以比较数是$E[i]-x+1$，它的含义是最大负载和$W_limit$的差距除以$m$，即在$E[i]$的基础上提前几个时刻来开始batch.

为了防止step 0时出现负载峰值，应该逐渐增加$W_{lim}$或者使用一个fixed $F$

#### Workload-balanced Hardware Selection

为了充分利用GPU和CPU，需要确定CPU的具体数量，不产生浪费。

$B$代表batch size,$P$代表CPU的数量，$T(B)$代表一个transformer block在GPU上计算S-part的时间，S代表用户设定的最大序列长度，L代表生成序列的预期延迟

假设pipeline中R-part的最大延迟和S-part相等，那么$N$ layers下生成一个token的latency为：

$$2NS\cdot T(B)\leq L$$

当Batch size越大时，throughput越大，但是$T(B)$也会越大。而且当B足够大时，带来的throughput就会越来越少，因此需要选择一个合适的B。

$E(B)$是GPU的throughput efficiency，当B很小时，$E(B)$增加的很快。当B足够大时，$E(B)$增加便会减小。

$$E(B) = \frac{B}{T(B)}$$

第三个约束条件是CPU的host-side memory capacity，需要考虑到KV cache的大小，C是每个CPU处理的最大token数，即满足：（1/2指的是上述pipeline减小到的最大workload）

$$\frac{1}{2}BS\leq CP$$

那如何确定P呢，假设一个CPU处理一个token的R-part的时间是R，需要满足：

$$\frac{BSR}{2P}\approx T(B) \Rightarrow P\approx \frac{BSR}{2T(B)}=\frac{1}{2}SRE(B)$$

一个简单分析，S-part和feature维度h成平方正比，R-part和h成正比，因此P和$1/h$成正比，当模型特征维度较大时，P通常会更小。







