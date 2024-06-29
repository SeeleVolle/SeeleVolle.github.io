### Preble

The first **distributed** LLM serving platform that targets and optimizes for prompt sharing. computation reuse and load balancing

+ 面对的是Long prompts and shorter generation的情况，prefill阶段更加重要
+ Prompt在请求之前有部分共享

Reuse the computation of shared prompt prefixes: 在处理分布式服务的时候，需要有state. 传统的方法存在一些问题：

+ 传统分布式系统通常将computation和state保存在单独的cluster中，从而允许独立的放置computation和data。但是推理中的注意力计算需要直接的访问KVs(State)，所以State placement需要和computation放在一起。
+ 传统分布式系统中一个可存储的对象的任何部分可以被缓存并被广播到所有服务器上。但是推理中只有matched prefixes能够被共享并被缓存，并且完整的prefix必须在同一个GPU上来进行高效的矩阵计算

+ 不同的Prompt能够形成静态的sharing-based prefix tree，但是在online场景中，随着请求的到达和离开，prefix tree的结构和每个树节点的load都在快速变化。LLM serving系统的性能除了请求负载还和prefill-decoding和平衡有关

key portion: 分析了现有的五个LLM-workload和data-center LLM request得到的一个结论，大多数的请求的prompt中有一部分和前一个请求具有不同的sharing features，并且比前一个请求的总长度要长，这个部分称为*key portion*

+ 在前缀树中定义为一条路径上深度最深的，满足token数比它所有predecessors' token数都多的节点，即the tree node with the longest tokens)

load cost: 为每张GPU计算长期GPU load和特定的请求负载，基于**token count**, **sharing features**, and **request count**来计算

+ GPU在运行当前scheduled请求和不久后的预期负载：
  + 使用最近的请求中的non-cached prompt prefill and decoding time来估计
+ GPU关于重新计算的负载
  + GPU需要腾出内存来执行当前请求
+ GPU上当前正在运行的请求负载

E2(Exploitation and Exploration): 提出的一个分布式的请求调度算法，能够基于GPU负载和prompt共享特性动态调整请求和状态(prefix)调度. 

+ 当shared prefix比剩余的token数更长时，会选择exploitation，即发送请求给缓存有prefix的key portion的GPU并重用计算结果，否则会选择Exploration，即基于负载成本计算从所有的GPU中选出最佳选项来进行prompt计算。

+ 基于贪心的调度算法，仅考虑优化key-portion，而不是整个prompt。
+ 基于vLLM和SGLang
+ 不会更改初始assignment后prefix的位置，Preble会检测load的变化从而将request进行重定向到负载轻的GPU上，Preble also supports autoscaling by replicating a key portion and its prefix on multiple GPUs？what is autoscaling
+ 基于以下设想：命中cached prefix的prompt当作解码阶段的计算，missed prompt当作prefill的计算，because of the high prompt-to-decoding token length ratio？所以将missed request发给那些heavily hit request GPUs来平衡prefill和decoding的计算负载
+ 使用优先级调度算法，优先级基于每个等待请求的prefix cache hit ratio，为每个优先级分配各自的请求配额。

目前LLM serving的一些工作：

+ Decoding-centric LLM serving: 优化scheduling, memory usage, GPU utilization，data parallelism和Model parallelism for multiple GPUs。
  + Orca, vLLM, AlpaServe
+ Prompt-Aware LLM Serving: 平衡prefill和decode之间的时间不平衡问题，chunked prefill是将prompt分块，将这个块和decoding request在一个iteration中的一个batch进行计算。separate prefill and decoding指的将两个过程分布到不同的GPU上进行计算。
  + SGLang, Hydragen

五种workload: Tool Use, Embodied Agents, Program Generation, Video Question and Answer, Long Document Question and Answer

四个Implication:

+ Optimizing prefill compu-tation can largely improve overall application performance, and imbalanced prefill and decoding computation features should be considered in LLM serving. 
+ Reuse computa-tion across shared prefixes can largely improve real work-loads’ performance and should be efficiently supported by distributed LLM serving systems. 
+ Identify-ing the key portion of prompts and optimizing the placement of requests according to their key portions is a viable way of reducing the complexity of scheduling while achieving good performance. 
+ An efficient LLM serv-ing system should consider complex, mixed-usage scenarios and factor in both load and prompt sharing variations.

#### Preble Design

<img src="../assets/image-20240530184818050.png" alt="image-20240530184818050" style="zoom:80%;" />

Preble: 基于E2算法的分布式调度系统。由两个调度器组成，一个是request-level global scheduler，另一个是per-model-instance iteration-level scheduler。前者用于调度请求到GPU上，后者用于已经调度到GPU上的请求

+ Prompt-Aware
+ model parallelism(Standard Tensor parallelism) and data parallelism

##### **E2 Global Scheduler**

**Global scheduler data structures**: 

+ global prefix trees(radix tree)，每个tree有一个不同的root(prompt的开头部分)，每个tree node是当前requests中的具有相同sharing property的a sequence of tokens，末尾的leaf node是未匹配到的剩余tokens.
+ tree node: 维护节点内的token数，存储当前node(KVs for the tokens in tree node)的GPU set，在历史窗口W中共享该节点的每个GPU的请求数。

<img src="../assets/image-20240530190144105.png" alt="image-20240530190144105" style="zoom:67%;" />

**Per-request scheduling policy**

对于一个request，我们首先在radix tree中进行prompt匹配，然后可以得到matched prefix的token数(re-computation)，和non-matched remaining prompt的token数(new computation)，两者进行对比，前者大就exploitation，否则就exploration. Exploitation的操作是将request分配给存有对应key portion的GPU，多个cache时选择负载最轻的GPU. Exploration的具体过程是使用三种cost来计算per-GPU cost，选择lowest load costs的GPU来执行。

GPU Load Cost Calculation: 分为三部分，L是不考虑当前请求的GPU’s overall load，M是保证当前请求k能够运行的GPU需要释放内存的potential cost， R代表actual cost

+ L Estimated: $L_i=\sum_{r\in W}(PT_r+DT_r)$

  对于每个GPU，选取最近的H个路由到该GPU的请求，对于H个请求中的每一个request r，用一个回归函数和不匹配的token数估计$PT_r$, 用另外一个回归函数和H个请求内平均Decoding时间估计$DT_r$。回归函数的获得是offline的，profiling for GPU type. 并且一个workload中request的生成token数较少而且比较相似，因此新request的decoding time和之前也比较类似。 

+ M Estimated: $M_i=PT_j\times N_j$, 前者是重新prefill evicted tokens所花的时间，后者是共享node j的请求数目.

+ R Estimated: $R_i$代表在每个GPU上实际运行当前请求k的cost，只包括prefill time，因为decoding cost已经在L中包含了

  

<img src="../assets/image-20240530191050257.png" alt="image-20240530191050257" style="zoom:80%;" />

**Post-assignment load adjustment**

作用：处理对于某个prefix来说，load和key portion发生变化的情况

+ 在GPU之间转移负载，即当最heavy的GPU的负载比最低多于一定倍数时(Google File System)就进行负载转移，这个$Th_{bal}$可以offline从GPU和LLM types中得出，这个转移的时刻是在分配前，即根据E2算法执行的结果进行修改
+ 在GPU之间将某个Prefix复制或者将其subtree进行split(请求的平均排队时间高于一个值时)

**Prefill-decoding balancing**

基于以下设想：命中cached prefix的prompt当作解码阶段的计算，missed prompt当作prefill的计算，所以一个request的所有prompt都命中的话就看做是Decoding，有很长的prompt未命中的话就看做Prefill. 

具体来说，对于一个需要Exploration的请求，如果一个GPU的Decoding-phase的负载非常重，就直接把这个请求给它。如果所有GPU的decoding-prefill balance都相近的话，就计算load-cost. 

**Global scheduler scalability**

+ 请求首先由并行的标记层进行标记，然后全局调度器生成异步请求处理程序来处理和调度请求
+ 请求处理中对radix tree的访问是不加锁的，更新tree node对应的GPU和request hit number时使用原子操作
+ 全局调度器维护每个GPU的当前负载，方便请求到达时更新
+ rebalance和eviction由额外的独立线程来进行，不影响主线程性能

##### Local Scheduler

**Local scheduler mechanism**

每个GPU上维护a request wait queue,a prefix radix tree, 以及每个tree node当前被共享的active request数目。请求到来时，就匹配local prefix tree并对对应的tree进行更新，并将请求加入到等待队列。每次iteration的batch，是通过priority-based algorithm从等待队列中选择，并对那些long and non-shared prompt进行chunk。如果当前GPU内存不够，就选取一些tree node进行丢弃，基于request access time(LRU)

**Waiting queue request ordering**

Priority-based wait queue scheduling policy: 创建P个优先级组，根据每个请求的cached token百分比将请求划分到不同的组里，scheduler会按照比例从各个优先级组选择。

使用三个指标评估：request per second, average request latency,  p99 request latency

