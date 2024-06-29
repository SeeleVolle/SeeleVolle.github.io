# DEFT: FLASH TREE-ATTENTION WITH IO-AWARENESS FOR EFFICIENT TREE-SEARCH-BASED LLM INFERENCE
总结：减少了KV cache和partial results的IO cost

+ IO-aware tree attention algorithm
+ tree-search-based tasks target

+ QKV Preparation and Attention Calculation(IO in KV cache, QK^T, Softmax)

Sequenced-based decoding

Tree-based decoding: 同时处理多个具有共同prefix的序列，tree中的节点可以在计算和内存中共享

+ 直接应用Sequenced-based带来的冗余: KV cache带来的memory storage, common prompts带来的computation, memory access

Tree attention: 用来处理并行decoding的场景，减少Attention计算时内核启动，计算和KV cache存储的overhead

+ 不足在partial result(QK^T)和KV cache都不是IO aware的

Two Key insights:

+ Query矩阵的IO workload和KV cache相比不值一提，Tree中每个节点的KV cache的长度远远大于Query的长度
  + 在KV-preparation中，将Decoding tree按照每个节点所拥有的tokens和KV cache进行拆分，然后将每个节点的KV cache与Decoding tree中共享它的所有Query分组
+ 在Tree-based attention中，多个query矩阵可以在计算中共享它们ancestor的KV cache，从而优化KV cache storage和减少IO
  + fused-kernel to get partial attention and conduct the tree-topology-aware global reduction

Decoding tree definition: Decoding tree的root节点是原来的prompt，每个non-root节点代表了生成token的一个序列，每个leaf node的最后一个token是下一次解码迭代的input token

Tree attention definition: $\text{Tree-Attention}(n)=\text{Attention}(P_{root \rightarrow n})$，即对于某个节点$u$中的token $n$，它的Tree-attention指的是$P_{root \rightarrow n}$路径上的所有Sequence attention output的总和，而$P_{root \rightarrow n}$指的是$P_{B_u}+s_{u,n}$，分别代表的是从根节点到节点u(不包含u)在内的所有token和节点u内从第一个token到token $n$的序列

**Deft Attention Kernel**

+ 三个需求：

  + Query(tokens)

  + KV cache of decoding tree

  + Tree topo(the topology of decoding tree to map Query and KV)

+ 三个部分：

  + Branch Controller
  + KV cache Manager
  + Sequence Tree Manger

两张核心图：

+ System Overview:

<img src="../assets/image-20240604111511926.png" alt="image-20240604111511926" style="zoom:50%;" />

+ Comparasion: 这张图讲的很清楚

<img src="../assets/image-20240604112235509.png" alt="image-20240604112235509" style="zoom:50%;" />



#### **Algorithm**

**QKV Preparation**: load Query, Key, and Value (QKV) into shared memory and group them logically to calculate attention

KV-Guided Tree Split strategy with tree-topology awareness: 即以每个node的KV cache来进行分组，消除了KV cache的IO cost，引入了额外的Q的IO，但由于在tree decoding的时候，KV的token length通常远远大于Q，因此Q的IO可以忽略不计。并且DefT不需要casual mask来进行之后的attention计算.

<img src="../assets/image-20240605133402029.png" alt="image-20240605133402029" style="zoom:50%;" />


**Attention Calculation**: apply attention algorithms to QKV groups for final attention results.

Tree-Topology-Aware Global Reduction Strategy：将每个QKV group进行分块，从而利用shared memory进行计算，然后增量的进行softmax reuction来重建attention。

<img src="../assets/image-20240605133414636.png" alt="image-20240605133414636" style="zoom:50%;" />

每个QKV group会交给一个线程块进行计算with flash attention，计算出partial attention后，将partial attention和query进行map，然后进行global reduction.

<img src="../assets/image-20240605115523939.png" alt="image-20240605115523939" style="zoom:50%;" />

与其他现有方法进行IO复杂度分析对比：

<img src="../assets/image-20240605115630471.png" alt="image-20240605115630471" style="zoom:50%;" />

