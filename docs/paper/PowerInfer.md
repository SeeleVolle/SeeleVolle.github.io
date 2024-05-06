# PowerInfer

解决模型参数过大的问题： 

+ Compression：quantization, distillation, pruning
+ Offload：将模型在Transformer这一层分到GPU和CPU之间，同时利用CPU的能力进行计算(llama.cpp)

Locality mismatch：硬件和LLM特性之间的局部性不匹配，指的是参数过多导致单张GPU不能装下从而利用自身(低容量)的高带宽

做了哪些工作：基于Power-law Activation和Fast In-CPU Computation

1. PowerInfer 提出了一种自适应方法，用于构建针对具有更高激活稀疏性和偏斜度层的小型预测器。这让原本占用大量GPU内存的online predictor的size减小同时保持精准度
2. PowerInfer设计了neuron-aware的稀疏运算符， 该operator可以和单个neuron直接交互，从而避免操作整个矩阵，以及特定稀疏格式的转化
3. PowerInfer 在offline阶段生成了neuron placement policy，通过一个评估标准将问题变为整数线性规划问题，标准考虑了神经元的激活频率还有CPU/GPU的带宽架构层次

Background:

+ Activation Sparsity：LLM的推理过程中，神经元的激活有很大的稀疏性，无论是在Attention还是在MLP中。可以在模型迭代过程中，提前基层预测神经元的激活情况。

+ GPU-centric offloading: 使用CPU内存存储超过GPU内存限制的模型参数，会有极高的Latency，因为CPU和GPU之间的PCIE传输比较慢

+ Hybrid offloading: 同样是在CPU和GPU之间分布模型参数，首先让CPU进行计算，再将结果发送到GPU去进行token生成，从而最小化数据传输，但是CPU占据了大量的数据处理负载，导致结果很慢

+ DejaVu：通过使用选择性处理激活神经元的方法来加速推理，但是bottleneck在于在运行时需要频繁的将激活的神经元从CPU传输到GPU，同样受到传输速率的限制，有极高的Latency

PowerInfer OverView：

> 将频繁激活神经元的权重矩阵预先加载到GPU上，其余保存在CPU上。在推理过程中，PowerInfer会跳过大多数不活跃的神经元，仅仅计算由在线预测器预测为活跃的神经元。由于预加载策略，已加载到GPU上的hot-activated neurons占总激活的主要部分，因此可以将大部分的计算任务分配到GPU上，而那些cold-activated neurons，则由CPU计算，避免权重矩阵的传递。

+ Offline phase(Profiler & Solver) -> Online(CPU executor and GPU executor)
  + Step1: Profiler会先从General dataset收集神经元在所有层上的激活数据，并由Policy Solver决定它是hot还是cold
  + Step2: Policy Solver会根据神经元的Impact和具体硬件情况来平衡workload，使用Integer linear programming来最大化GPU上神经元的对结果的总Impact
  + Step3: Engine在处理用户请求之前，会根据offline solver的输出将两种不同的神经元分配给各自的计算单元
  + Step4: 运行时Engine会创建CPU和GPU Executor，去管理CPU和GPU计算，同时进行online prediction并绕过那些不激活的神经元

Adaptive Sparsity Predictors:

+ DejaVu使用两个分隔的Predictor来预测self-attention和MLP blocks中神经元的激活
+ Accuracy受两个因素影响：Activation sparsity和internal skewness，在transformer layer有高激活稀疏性和高内部偏斜度的情况下，小规模的预测其也能有很好的精准度
+ PowerInfer设计了non-fixed-size predictors的迭代训练方法，对于每个Transformer layer，首先根据该layer的sparsity profile结果建立基础的baseline model size，然后根据该层内Activation skewness对准确性影响进行size的调整。对于MLP Layer，主要根据skewness对隐藏层的dimension进行调整，高skewness就减小隐藏层维度，低就增加将精准度保持在95%。

Neuron Placement and Management

+ 对于每层来说，有很多权重矩阵，PowerInfer会把对应的Neuron放到GPU或者CPU上
+ CPU和GPU上都有一个Neuron Map，记录了Neuron在矩阵中的原始位置，便于计算时找到相应位置

Hybrid Execuation Model:

+ 在进行Inference之前，建立了一个计算图并存储到CPU memory中的global queue上，计算图中的每个节点代表LLM推理中的一个operator。
+ 由主机上的线程执行CPU executor和GPU executor，它们从全局队列中获取operator检查依赖关系，并将operator进行分配。计算结果的合并在GPU上进行
+ 正常情况下，CPU和GPU上都有激活神经元时结果需要同步，但当CPU上没有时，就不用同步了

Neuron-aware Operators:

+ 首先确定神经元的激活状态，如果预测是激活那么就与参数矩阵的相应行/列一起处理
+ 在GPU上，所有的线程块可以同时检查神经元的激活状态，并在激活时进行计算。在CPU上，将operator分成多个小批次进行并发激活检查，每个核心仅仅处理其对应批次中的激活神经元。

Neuron Placement Policy:

+ 考虑了Neuron的激活频率，通信开销，计算单元的计算能力(内存和带宽等，之后再Integrate)，建立了Neuron Impact从而对Neuron的激活信息进行了建模

+ Offline Profiling: 从多个常用数据集上获取LLM中各个神经元的runtime inference data，在每个Transformer Layer后添加一个Monitor kernel来精准评估激活信息，并在GPU上建立一个neuron information table来记录每个neuron的activation count.

+ Impact Metric: 用激活频率来作为Impact Metric，因为frequency反应了runtime behavior

+ Model of Placement: 

  + cumulative impact:   $t_i=\sum_{e \in N}a_{ie}*v_e, \forall i\in{GPU}$, $\sum_{i\in  U}a_{in}=1, \forall n \in N$

  + 目的是要**Maximize前一个式子**，其中$a_{ie}$指的是在GPU i上的所有神经元，后面那个式子的意思是对于某个n来说，在所有设备i上存在的概率和为1。除了Maximize前一个式子还要考虑之前所说的通信开销和硬件情况

  + Communication constraint: 

    为了减少通信开销，GPU上神经元的preload需要有一个minimum（每一层的$C_l$），确保了$C_l*T_l^{GPU}+T_{sync}\leq C_l *T_L^{CPU}$，用访问一次神经元的所有权重的时间来估计神经元的计算时间$T_{ij} = M_i/Bandwidth_j$，而在batch size较小的情况下，每个层内数据传输范围保持一致，所以同步成本相同，所以用将Tsync描述为单个层内通信实例的profiled overhead

  + Memory Constraint: 

    $\sum_{n \in N}a_{jn}·M_n<MCap_j, \forall j\in U$，指的是对于CPU和GPU来说，其上面的神经元所占memory大小需要小于对应的处理单元。此时对于某层l来说，也需要满足GPU上的神经元个数大于$C_l$或者为0

+ Integer Linear Programming:

  + 将每一层的神经元聚合成batch并进行collective placement analysis，也就是将具有相似impact的neuron封装成一个group，以group为单位来进行ILP

相关研究：

+ LLM Activation Sparsity:  DejaVu, PIT, brainstorm
+ LLM Weight Sparsity: Model Pruning(SparseGPT, Wanda), SparTA, FLash-LLM
+ Speculative LLM Inference: Speculative inference, Speculative decoding, SpecInfer
+ LLM-Specific Serving Optimizations: vLLM, Orca


