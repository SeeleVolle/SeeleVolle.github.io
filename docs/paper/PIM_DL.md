# PIM-DL: Expanding the Applicability of Commodity DRAM-PIMs for Deep Learning via Algorithm-System Co-Optimization

三点核心贡献：

+ PIM-DL: 提出了第一个在DRAM-PIMS上的DL框架，使用 LUT-based deep-learning paradigm

+ 提出了增强的LUT-NN算法来进行模型校准
+ 设计了LUT-NN在DRAM-PIM上的映射，并对数据流进行定量建模。提出了一个Auto-tuning框架来优化在不同DRAM-PIM平台上的映射

#### eLUT-NN:  

对于某一层，**different input activation matrices**的features具有块语义相似性，所以可以用这些称为**centroids**的特征来近似整个矩阵，而原本的GEMM操作可以转化为centroids和weight的乘法。

通过提前得到centroids，那么就可以提前计算出centroid和对应weight的值(称为partial-sums)并存入到_look-up tables_。在Inference过程中，我们仅需根据和当前Input最接近的centroids' indices从LUTs中取出Pre-computed data，再进行reduction就可以得到原先GEMM的结果。

##### **LUT-NN Conversion**

<img src="../assets/image-20240511153857317.png" alt="image-20240511153857317" style="zoom:80%;" />


假设Activation Matrices大小是$M\times H$。

1. 沿着$H$维度，我们将其分成$1\times V$的子向量，因此每一行会得到$\frac{H}{V}$个子向量。然后，对这M个向量(within the column, then what's mean across activation matrices)执行K-means算法，得到四个centroid向量，称作codebook。
2. 将Weight matrix($F\times H$)沿着$H$维度分解成$F\times V$的子向量，然后与对应的codebook中的向量进行inner-products(向量点乘)，四个向量会产生四个LUT，大小是$F\times \frac{H}{V}$

##### **LUT-NN Inference**

<img src="../assets/image-20240511160344980.png" alt="image-20240511160344980" style="zoom:80%;" />


1. Inference过程中，对于Input activation matrix，将其分解成$\frac{H}{V}$个$1\times V$的tiles，将每个tile中的vector与对应的codebook中的centroid进行点乘并找到最小值(closest L2-distance with inner products)。
2. 根据最小值对应的centroid的index，从对应的LUT(codebook中每个centroid对应一个LUT)中读出一列数据。
3. 对于矩阵的每一行，从LUT中找到的$F\times 1$的向量进行**accumulate**得到本行最终计算结果。遍历所有行将所有行的结果拼接起来，就得到了$F\times M(N)$的结果

计算变化：

+ GEMM: $N\times H$的矩阵和$H\times F$的矩阵相乘，需要$2NHF$的计算，$NHF$是乘法，$NHF$是加法
+ LUT-NN: 
  + Index Calculation: $3\times N\times \frac{H}{V} \times V \times CT=3NH\times CT$，加法乘法和Min
  + Result Accumulation: $N \times \frac{H}{V}\times F$, 加法操作

#### **PIM-DL Framework**

从下图中可以看到，PIM-DL Framework由三部分组成，LUT-NN Converter，Auto-tuner和Inference Engine组成。解决了当前的LUT-NN替换精度不高，不同硬件特性不同以及没有同时支持CPU/GPU和DRAM-PIM DL serving框架的问题

<img src="../assets/image-20240511162458821.png" alt="image-20240511162458821" style="zoom:80%;" />


##### **LUT-NN Converter**

eLUT-NN校准算法: High data efficiency and High model accuracy，使用两个技术来进行模型校准，High data efficiency and  High model accuracy

+ Reconstruction Loss for computation approximation
+ Straight Through Estimator for gradient propagation.

**Reconstruction Loss for computation approximation**

Reconstruction Loss收集了所有被替换Layer的error和原始的model loss来构建LUT-NN's 校准损失L: 

$L=\text{Model Loss}+\beta\sum_{l\in L}||\hat{A_l}W-A_lW||^2$，其中$A$代表原始矩阵，$\hat{A}$代表用centroid替换后的矩阵，$l$代表layer $l$.

+ Enable direct gradient propagation
+ Enable the centroids to learn accurate representations of activations

**Straight Through Estimator for gradient propagation**

由于centroid clustering and table-lookup operators不可微，所以使用STE来估计梯度和反向传播，F是Layer Input的生成函数。

STE：近似地处理离散变量的梯度，在前向传播过程中使用离散变量的值，在反向传播的过程中将离散变量的梯度设置为单位矩阵。这样在传播过程中，梯度会被无修改的传递到上一层。

$\dfrac{\partial L}{\partial F}=\dfrac{\partial L}{\partial \hat{y}} \cdot\dfrac{\partial \hat{y}}{\hat{A}}\cdot \dfrac{\partial\hat{A}}{\partial A} \cdot\dfrac{\partial A}{\partial F}\approx \dfrac{\partial L}{\partial \hat{y}} \cdot\dfrac{\partial \hat{y}}{ \hat{A}}\cdot\dfrac{\partial A}{\partial F}$，其中$\hat{y}=\hat{A}\cdot W$，是本层的输出。



##### PIM-DL Engine

<img src="../assets/image-20240511192742121.png" alt="image-20240511192742121" style="zoom:80%;" />

PIM Operators:

+ PIM Kernel: 在host上触发PIM module来执行workload
+ PIM Binary: 在PIM module中describe offloaded workload

Host Processor通过PIM Driver来控制DRAM-PIMs的执行，在Transformer的执行过程中，将QKV Projection，O Projection， FFN这些线性层都替换为LUTs。Attention在host上执行，其余的Add,Norm, GeLU都在PIM上执行

#### Hardware Mapping and Optimization

LUT-NN's mapping strategy and PIM-DL Auto-Tuner: 产生input LUT-NN和target hardware platform的best mapping parameters

DRAM-PIM架构图：在每个PIM module中，有很多分布式的计算节点。每个节点由一个Processing Engine和local memory banks组成。核心在于所有的PE可以同时进行内存访问，所以内存带宽很高，相较于一般的计算密集型设备

执行PIM workload分为三步：host processor准备输入数据并发送到PIM modules，准备好后host会启动PIM kernel，当所有的PE执行完Kernel execution后，host会从所有的PIM module获取结果。

不过DRAM-PIM有一些缺点：

+ Constrained Host-PIM communication：所有的PE共享一条数据总线，所以每个PE的传输速率有限，所以尽可能的使用broadcast
+ No direct inter-PE datapath: 由于on-chip的routing resources有限，所以inter-PE之间没有直接的数据通路，所以得尽量避免inter-PE communication
+ Load-balancing problem: 短板效应
+ 
<img src="../assets/image-20240605153152800.png" alt="image-20240605153152800" style="zoom:100%;" />


**LUT-NN的两个operato**r: CCS(closet centroid search)和LUT(table lookup operator)，CCS在host上计算，LUT在PIM上计算

+ LUT: 分为两步，第一步是将LUT划分为sub-LUT workloads，第二步是在所有的PE上并发的启动所有的kernel

  + Sub-LUT Partition:  注意图中Lookup Table的CB和CT进行了Transpose，在图中，对于CCS获得的Index matrix和Pre-computed LUT，分别沿着N和F两个维度进行PE group之间的均分。$i$-th PE group负责计算$i$-th index tile，即图中的红色部分。PE group中的$j$-th PE负责计算对应index tile和$j$-th LUT tile的table lookup。

    $t_{sub-lut}=t_{index}+t_{lut}+t_{output}$

    这样做有一些好处：

    + 在同一个PE group中的所有PE共享同一个index tile，所有的group中的$j$-th PE共享同一个LUT tile，充分了利用了host-PIM带宽
    + CT维度没有分割，避免了inter-PE communication
    + 均匀分割保证了workload的均衡

    <img src="../assets/image-20240607104321494.png" alt="image-20240607104321494" style="zoom:100%;" />
    
  + Micro Kernel Execution: 对于Output中的每个$N_{s-tile}\times F_{s-tile}$的tile，为了利用PIM上的on-chip buffer，继续进行分块，对于$N_{s-tile}\times CB$的index matrix，在两个维度上继续进行分块，称为M-tile。每个PE加载一个index Mtile，然后通过LUT产生一个output Mtile，对于每个Mtile, 需要将所有同行的$N_{m-tile}$加起来得到最终的output Mtile

    $t_{micro-kernel}=t_{transfer}+t_{reduce}$

    <img src="../assets/image-20240607110516375.png" alt="image-20240607110516375" style="zoom:80%;" />



**PIM-DL Auto-Tuner**

主要找的是四类参数：

+ Sub-LUT Tiling Factors: $N_{s-tile}$和$F_{s-tile}$
+ Micro Kernel Tiling Factors: $N_{m-tile}$, $CB_{m-tile}$, $F_{m-tile}$
+ Tile Traversal Order: 即上述的三个Mtile factors的遍历顺序
+ LUT load scheme：三种LUT matrix的load scheme
  + Static Load：当LUT的Mtile小于on-chip buffer size时，就将整个Lookup Table tile($CB_{s-tile}\times CT \times F_{s-tile}$)加载到buffer中
  + Coarse-grained load: 考虑到每个Index会从$CT$ candidates中选取一个，所以每次加载$CB_{load-tile}\times CT \times F_{load-tile}$大小的LUT tile到buffer中
  + Fine-grained load: 在这种方法中采取on-demand的方法，具体是每当处理一个新的index时，就沿着F维度加载$F_{load-tile}$的elements，当并行处理多个read-request时，可以为每个线程keep一个$F_{load-tile}$大小的Buffer

<img src="../assets/image-20240607113317402.png" alt="image-20240607113317402" style="zoom:80%;" />


