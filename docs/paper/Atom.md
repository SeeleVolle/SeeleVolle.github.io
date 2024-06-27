# ATOM:LOW-BIT QUANTIZATION FOR EFFICIENT AND ACCURATE LLM SERVING: 

Background:

量化的步骤：

+ 确定量化参数，包括scale和zero point
+ 计算量化后的tensor

uniform asymmetric quantization：

$s= \dfrac{max(X)-min(X)}{2^n-1}\cdot c, z=\lfloor\dfrac{-min(X)}{s}\rceil,\overline{X}=clamp(\lfloor\dfrac{X}{s}\rceil,0,2^n-1)$

其中$s$是scale, $z$是zero point，$\overline{X}$是量化后的Tensor，$X$是Input Tensor，clamp是一个限制函数，将$\lfloor\frac{X}{s}\rceil$限制在$[0,2^n-1]$之间

量化后的乘法：$W\cdot X=s_W(\overline{W}-z_W)\cdot s_x(\overline{X}-z_x)$

uniform symmetric quantization:

$s= \dfrac{2*max(|X|)}{2^n-1}\cdot c,,\overline{X}=clamp(\lfloor\dfrac{X}{s}\rceil,-2^{n-1},2^{n-1}-1)$

dynamic vs static: 

动态量化使用的是Inference时实时统计信息，而静态量化使用的是校准数据

量化的granularity:

+ Per tensor: 对于整个tensor做量化

+ Per channel：对于Tensor的一行或一列做量化

+ Per group：在Tensor的一行中分为不同的组，对这些组做量化

  

核心量化设计：

+ 采用混合精度量化，保留一小部分显著的actiavtions和weights来维持精度
+ 细粒度的group quantization权重和激活
+ 动态地量化activations的参数来捕获每个输入的分布

为了利用硬件特性：

+ 对于混合精度操作，重新排序权重和激活
+ 融合量化和重排序操作来消除overhead
+ 将outliers量化为8位，来保持精度和效率之间的平衡
+ 量化KV-cache到低精度来减少内存移动

Atom LLama架构：

<img src="../assets/image-20240509100411900.png" alt="image-20240509100411900" style="zoom:75%;" />


量化流程如下：

<img src="../assets/image-20240508092029094.png" alt="image-20240508092029094" style="zoom:75%;" />

**Mixed-precision quanzization**

采用混合精度的分组量化，outliers采用INT8量化，其余采用低精度量化

在量化时，将分散的outliers放到weight和activation矩阵的末尾。

Weight matrix的重排序使用校准数据离线进行，只用进行一次。Activation Matrix的重排序需要在线进行，融合到了在其之前的Operator中来减小overhead.

**Fine-grained group quantization**

进一步提升准确率

Fused GEMM operator：首先对于每个group使用TensorCore进行的低精度的乘法计算，然后将结果还原并进行FP16的加法得到最终结果。即将Dequantization和原先的MMA算子进行了融合。

<img src="../assets/image-20240509100350959.png" alt="image-20240509100350959" style="zoom:75%;" />

**Dynamic quantization process** 

对于激活矩阵，采用动态量化，实现类似于ZeroQuant(?).

使用对称量化，选择了一个好的clip threshold

在权重矩阵的量化时使用GPTQ来提升准确率

**KV-cache quantization**

使用非对称量化，在一个head内进行量化

在Load时使用low-bit的量化，而在进行FP16计算之前进行还原