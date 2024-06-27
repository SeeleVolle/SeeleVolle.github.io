# LUT-NN: Empower Efficient Neural Network Inference with Centroid Learning and Table Lookup

具体包括两个方面：

+ 通过反向传播的可微质心学习，采用三个近似级别来减少精度影响
+ Table Lookup Inference Execuation，考虑不同级别的并行性，内存访问和专用硬件

存在的问题：精度下降比较严重，原因在于采用的PQ(Product Quantization)和DNN的优化目标不同，前者在于最小化centroid和vector之间的error，而后者在于最小化final loss function，两者之间没有直接联系。

关键在于将model loss反向传播传递给每个operator，然后通过传来的梯度调整centroid来降低model loss。但是找centroid的过程是不可微的，本文提出的解决方法是：three levels approximation by centroids to inference by three methods

+ 使用soft-PQ来允许梯度计算和迭代调整centroid，即对于argmax做了近似替换：在反向传播时使用softmax来替换，前向传播依旧不变(argmin)
+ 使用learned temperature method来学习每个operator中argmax的超参数，平衡精度下降和收敛速度
+ 使用scalar quantization，对于Pre-computed lookup tables，使用标量量化来减少存储需求，在前向传播中使用quantized tables，反向传播中使用真实table

Product Quantization: 相较于上文，这里提供了一些表达式

+ Centroid learning： 目标是最小化$argmin \sum_c \sum_i ||\hat A_i^c-P_k^c||^2$，
+ Sub-vector Decoding: $g^c(a^c)=argmin_k||a^c-P_k^c||^2$，这就是对于某个sub_vector的编码函数，实际在前向传播过程中将一个sub-vector用最近的centroid进行了编码
  + Hash based encoding: 为了减小encoding的cost，采用hashing方式来编码，将某个sub-vector丢到one of K buckets中

Construct lookup Table:

+ 即element-wise multiplication，Codebook中的每个元素和对应Weights中的值相乘得到最终的table
+ 对于Weight中的某个元素$b^c$: $h^c(b^c)=[P_0^c\cdot b^c,P_1^c\cdot b^c, \cdots,P^c_{K-1}\cdot b^c ]$
+ MM近似：$a\cdot b=\sum_ca^cb^c\approx\sum_cg^c(a^c)\cdot h^c(b^c)$,  $g^c(a^c)=onehot(argmin_k||a^c-P_k^c||^2)=(0,...,0,1,...,0)$

<img src="../assets/image-20240607153247986.png" alt="image-20240607153247986" style="zoom:80%;" />


[量化索引：Scalar Quantization标量量化 ](https://yongyuan.name/blog/scalar-quantization.html)

> eLUT-NN: 在PIM-DL中是采取了Reconstruction Loss，收集了所有被替换Layer的error和原始的model loss来构建LUT-NN's 校准损失$L$, 相较于本文算法使用的calibration dataset更少并且可以替换所有的Layers.
>
> $L=\text{Model Loss}+\beta\sum_{l\in L}||\hat{A_l}W-A_lW||^2$，并通过STE来进行梯度传播，避免了K-means不可微的问题

#### Differentiable Centroid Learning
<img src="../assets/image-20240607180021495.png" alt="image-20240607180021495" style="zoom:80%;" />

**Backpropagation through soft-PQ**

本来在反向传播过程中，encoding使用的是argmax，改用可微的softmax，$\hat g^c(a^c)=softmax(-||a^c-P_k^c||^2/t)$

encoding的实质过程就是将一个确定的onehot sub-vector转换为probability vector，最大的概率代表最近的centroid

对于一个Vector，softmax会计算K个distance，distance越小，softmax计算出的可能性就越大（加了负号）。

此时$a^cb^c=\hat g^c(a^c)\cdot h^c(b^c)-sg(\hat g^c(a^c)\cdot h^c(b^c)- g^c(a^c)\cdot h^c(b^c))$, $sg$的作用是选择使用softmax还是argmin, 使用argmin时采用onehot，即在计算原始矩阵和codebooks之间的距离时选取distance最小的centroid置为1，然后从LUT中选取列中对应范围内值为1对应的那个元素作为结果，即对于一个$V\times1$的tile，需要在列中间断的选取F个元素，F是Weight matrix的row维度

**Learned Temperature  Method**

$softmax(x)_i=\dfrac{exp(x_i/t)}{\sum_{k=1}^Kexp(x_k/t)}$，t越趋近于0，结果越趋向于onehot(argmax(x))，t越趋近于1，结果越趋向于$1/K$即均匀分布，t越大方差越小，但是approximation error越大，所以需要一个合理的$t$, 这个t也会在反向传播中通过学习得到

**Scalar quantized lookup table**

前向传播时使用quantized value，反向传播时使用real value，从而可以在反向传播时对LUT进行调整

#### TABLE LOOKUP INFERENCE DESIGN
<img src="../assets/image-20240607185749516.png" alt="image-20240607185749516" style="zoom:80%;" />


具体分为三个部分：Closed Centroid Search, Table read and Accumation

**Closed Centroid Search**

在CCS中，由于input sensor的行数通常远远大于列数，因此$N\gg V \ \text{and}\ N\gg K$, V和K分别是sub-vector length和codebook number，所有$operation \ intensity = \frac{2NVK}{NV+VK+NK}=\frac{2}{1/K+1/V}$，而正常情况下K和V都很小，因此operation intensity很小，那么distance computation就变成了一个memory-bound的问题。

设计了一个centroid-stationary algorithm，作用是将centroid matrices尽可能长时间的保存在register和cache中，通过为每个codebook在cache中保存K个centroid在inner loop中，使得只用从DRAM中读取这些centroid一次。

由于在codebook中比较找出nearest centroid是一个data-dependency的过程，因此难以并行。但是可以将codebook划分为多个sub-codebook，然后在每个sub-codebook内以及sub-codebook之间进行比较，就可以做到并行。

**Table Read and Accumulation**

使用向量化指令加速table read(SIMD SHUFFLE)，来减少table read的overhead

 为了尽可能的使用SIMD lanes(即SIMD instruction中并行处理单元的数量)，K=16，并且首先用INT16对结果进行accumulate，然后再使用INT32避免溢出	