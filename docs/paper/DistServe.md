# DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving

简介：将Prefill和Decoding计算放到不同的GPU上，从而消除了prefill-decoding coupled的推理，并单独为每个阶段优化资源分配和并行策略，并考虑了两个阶段的通信开销。

稍微展开，用instance来描述管理完整模型权重复制的资源的一个单元，即有prefill和decoding两种instance，两者分别管理自己的权重备份并进行单独的计算，prefill instance会在生成第一个token后，把中间生成的kvcache传递给decoding instance

一个instance可能会被分配到多个GPU上执行，每个instance的resource allocation和parallelism strategy都不同，由于decoding instance的GPU利用率较低，所以可能会将多个prefill instance和一个decoding instance分配在一起来提高GPU利用率

 Motivation:

+ 在prefill-decoding interference中，prefill step通常要比decoding step花费时间更多，因此decoding通常会被prefill所延误。即使将两者分开来调度，由于计算资源的竞争，decoding task由于逐渐增加的queuing delays(prefill tasks)也会变慢。即使优先调度两者中的任何一个也不能很好的满足两者的延迟要求
+ 由于prefill和decoding的计算特性，两者对于延迟要求不同，所以可能适合不同的并行模式。

LLM Serving常用的优化：

+ Batching: 将new request的prefill阶段和on-going request的decoding阶段进行并行，需要考虑TTFT和TPOT的平衡，现在的一种先进策略是将Prefill进行segmenting，然后attaching decoding jobs，但是实际上是将TTFT的部分换成了TPOT。
+ Model parallelism: 模型并行通常是intra/inter-operator parallelisms，Intra-operator parallelism主要指的是将计算密集型操作分配到多GPU上执行，降低了execution time，缺点在于high inter-GPU communication required。Inter-operator parallelism主要指的是将LLM layer分成多个stage，然后每个stage分配给一个GPU执行来组成pipeline，系统的速率基本呈线性拓展，缺点在于增加了execution time. 本文提出additional benefit: 减少了prefill和decoding phase的queuing delay

#### Tradeoff Analysis

放在最前面，当model可以装进一张GPU时，replication是一个非常好的选择，即每张卡有一个模型的副本

**Prefill Instance**

+ Batching: 这是一个compute-intensive的过程，当GPU已经成为compute-bound后，batch只会增加总处理时间。所以提前对特定的LLM和GPU进行profile，得到input length的threshold $L_M$，当低于这个阈值时才会做batch

+ Parallelism: profile后发现prefill phase表现的比较像一个M/D/1 queue，比较intra和inter parallelism，在arrival rate较低的时候，intra策略的TTFT更低，随着rate和queuing delay的增加，inter的TTFT更低。

  + 在single-device without parallelism的情况下，

    $Avg\_TTFT=D+\frac{RD^2}{2(1-RD)}$，其中D是请求执行时间，R是泊松到达率(单位时间内请求到达的平均次数)，第二个式子代表的是排队延时

  + 在2-way inter-op parallelism的情况下，即两个阶段分配到不同的GPU上进行，假设最慢的stage花费$D_m$的时间，那么$D\approx D_s\approx 2\times D_m$

    $Avg\_TTFT_{inter}=D_s+\frac{RD_m^2}{2(1-RD_m)}= D+\frac{RD^2}{4(2-RD)}$

  + 在intra-op parallelism的情况下，即将MM等计算分配到多GPU上进行，由于通信的overhead，$D\approx D_s=\frac{D}{K}$，其中$1<K<2$，表示并行策略带来的speedup coefficient

    $Avg\_TTFT_{intra}=\frac{D}{K}+\frac{RD^2}{2K(K-RD)}$

  + 结论：当TTFT的SLO要求较高的情况下，intra-op parallelism比较有效，并且K越低TTFT越低，而speedup coefficient K主要受input length，model architecture, communication  bandwidth, placement的影响

**Decoding Instance**

+ Batching: 这是一个memory-bound的任务，在当前的colocated系统中，增加decoding batch size会带来延迟的增加，因为sharing GPUs会在两种任务之间产生竞争，所以通常需要在TTFT和TPOT做一个权衡。而将两者分离，就可以将多个prefill instance分配给一个decoding instance，从而可以在decoding phase中在特定GPU上增大batch size，而不用牺牲TPOT
+ Parallelism: 在进行phase分离后，decoding阶段的batch-size主要受限于GPU memory capacity, 因为GPU需要为所有的active requests保存KV cache等中间激活值。通过model parallelism或者 advanced memory management techniques，可以scale the decoding batch size，当扩展到一定值时，就成了compute-bound的任务。
  + 对于Intra-op parallelism来说，随着GPU数量的增加，考虑到通信开销和利用率下降，Throughput增加和Latency下降的收益逐渐递减
  + 对于Inter-op parallelism来说，throughput几乎是线性增加，但是TPOP较高
  + 结论：当TPOT的SLO要求较高的情况下，仍然使用Intra-op parallelism。

**Practical Problems**

+ Variable prefill length：在实际的LLM application中，每个request的prefill length是不同的，那么在进行inter-op parallelism时，pipeline bubble就会不可避免的产生，从而导致延迟的上升
+ Communication overhead：主要是prefill instance将KV cache传输给decoding instance的传输延迟，和Input length以及arrival rate成正比。所以需要考虑prefill 和decoding instance的placement，两者之间的传输延迟是不可忽略的。

#### DistServe Method

通过考虑model architecture, workload characteristic, latency requirements, and SLO attainment target，会确定如下两点

+ Prefill和Decoding instance的并行策略
+ 两种instance的deployment number和placement on physical cluster

**High Node-Affinity Cluster**

在装有IB的高传输性能集群上，KV caches的传输延迟带来的开销几乎可以忽略，所以prefill和decoding instance可以任意放置而没有其他的限制，所以使用一个two-level algorithm，首先分别优化prefill和decoding instance的并行配置，以获得phase-level optimal的per-gpu goodput，然后，使用replication来match overall traffic rate.

Replication：指的是复制之前的prefill和decoding instance的配置到所有节点上

<img src="../assets/image-distserve_1.png" alt="image-distserve_1" style="zoom:80%;" />

在该算法中，直接遍历了所有的并行配置，每次通过模拟器在计算当前策略的prefill和decode的goodput，并得到最终的最佳配置。复杂度是$O(NM^2)$

如何Build Simulator: 通过分析预填充和解码阶段的FLOPs和内存访问次数，并使用latency model来近似推理执行时间

**Low Node-Affinity Cluster**


<img src="../assets/image-distserve_2.png" alt="image-distserve_1" style="zoom:80%;" />

在算法2中，考虑了单个节点的GPU的内存大小，通过将模型中不同的Layer组合成stage，并将每个instance切分成不同的segment(maintaining one inter-op stage)，那么就可以将Prefill和decoding的segment划分到同一个节点内，让激活值的传输仅使用NVLink。在节点内，对于同一个instance的segment，采用相同的并行和资源分配策略。

在该算法中，首先枚举所有的inter-op parallelism degree(the number of segments)，对于每个segment，再枚举intra-op config，最后模拟计算各自的goodput并最终replicate

<img src="../assets/image-distserve_3.png" alt="image-distserve_1" style="zoom:80%;" />

**Key Enhancement in Runtime**

+ Reducing pipeline bubbles: 为了减少non-uniform prompt length所带来的pipeline bubbles，通过每个batch所产生的new token来估计一个batch的真实运行时间。对于prefill instance，profile目标模型和GPU来得到能最大限度满足GPU的序列长度，在分配prefill batch的时候尽可能接近这个值。而对于decoding instance，则直接将该序列长度作为最大的batch size。

+ Combat busrtiness: 当workload出现Burst时，会造成大量的KV cache传输，为了避免潜在的memory overload，采用了一种'pull'的方式来进行KV cacahe transmission。具体来讲，采用的是decoding instance从prefill instance fetch kvcache as needed。并使用prefill instance的GPU memory作为queuing buffer。

+ Replaning: 采用periodic replanning，定期检查workload pattern的变化情况并作出调整。通过一个workload profiler来监控Key parameters的变化情况，例如the average input and output length, the average arrival rate. 
  