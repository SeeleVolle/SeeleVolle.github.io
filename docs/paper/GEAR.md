# GEAR:An Efficient KV Cache Compression Recipe for Near-Lossless Generative Inference of LLM:

Generative Inference with Approximation Error Reduction

三个核心部分：

+ 使用uniform quantization来量化具有similar magnitude的主要部分(outliers以外的部分)，4 bit
+ 使用低秩矩阵来近似量化残差
+ 引入一个稀疏矩阵，该矩阵由可忽略不计的large magnitude条目组成，以弥补这些异常值引起的单个误差

Minimize如下的误差：

$min_{\hat D,L,S}||X-\hat D-L-S||_F$, 其中$X$是$K_t,V_t$中的一个tensor，$\hat D$是quantized matrix，$L$是低秩矩阵，$S$是稀疏矩阵。

那么如何确定各个矩阵呢：

**Outlier-reduced quantization**

在进行对$X$的量化前，取$\frac{s}{2}\%$的最大和最小值出来，以full precision的形式加入稀疏矩阵$S=Filter_s(X)$，所以$S$和$\hat D$可以如下表示：

$Filters(X)_{ij} = \begin{cases}    X_{ij} & \text{if } X_{ij} \text{ in top or bottom} \\    0 & \text{otherwise} \end{cases}$

$\hat D=Quant_b(X-S)$

**Low-rank approximation**

**线性代数**

SVD分解：矩阵的奇异值分解

对于一个秩为 r 的矩阵 $\pmb{A}_{m \times n} $，必存在 m \times m 的正交矩阵 $\pmb{U}_{m \times m}$ ， $n \times n $的正交矩阵 $\pmb{V}_{n \times n} $，$ m\times n $的矩阵 $\pmb{\Sigma}_{m \times n} $，使得 

$\pmb{A}_{m \times n} = \pmb{U}_{m \times m} \pmb{\Sigma}_{m \times n} \pmb{V}^T_{n \times n} = \pmb{U}_{m \times m} \begin{pmatrix} \pmb{D}_{r \times r} & \pmb{O} \\ \pmb{O} & \pmb{O} \end{pmatrix}_{m \times n} \pmb{V}^T_{n \times n} \\\\ \tag{9}$

其中， $\pmb{D}_{r \times r} = \begin{pmatrix} \sqrt{\lambda_1} \\ & \sqrt{\lambda_2} \\ & & \ddots \\ & & & \sqrt{\lambda_r} \end{pmatrix}_{r \times r} $， $\lambda_1 \geq \lambda_2 \geq... \geq \lambda_r >0 $为 $\pmb{A}^T\pmb{A}$ 的 r 个**非零特征值**（从大到小排列）。 $\pmb{A}^T\pmb{A}$ 的 r 个**非零特征值开根号** $\sqrt{\lambda_1}, \sqrt{\lambda_2} , \cdots, \sqrt{\lambda_r} $（从大到小排列）称为 $\pmb{A} $的**正奇异值**，$\pmb{A}^T\pmb{A}$的 n 个**特征值开根号** $\sqrt{\lambda_1}, \sqrt{\lambda_2} , \cdots, \sqrt{\lambda_r},0_1,0_2, \cdots,0_{n-r} $（从大到小排列）称为$ \pmb{A}$ 的**奇异值**。



QR分解：矩阵的正交三角分解

设$ \pmb{A}_{m \times n} $的秩为 n ，则 $\pmb{A} $可以**唯一**地分解为

$\pmb{A}_{m \times n} = \pmb{Q}_{m \times n} \pmb{R}_{n \times n} \\\\ \tag{8}$

其中， $\pmb{Q}_{m \times n} $是**标准正交向量组矩阵**，$ \pmb{R}_{n \times n} $是**正线上三角阵。**

----



对于residual $R=X-(\hat D+S)$，通过观察到残差矩阵的值在奇异值的index增大的时候有明显的下降，可以推测出残差中存在coherent component，并由top singular values/vectors组成。通过这些部分，可以有效的对残差$R$进行近似，也就是：

$L=AB^T=SVDSolver_r(R)$，其中$A\in R^{n\times r},B \in R^{d\times r}$，通过一个算法来得到**top-r singular values/vectors** $\sum^{k}_{i=1}\sigma_iu_im_i^T$，其中$\sigma_i$是$R$的奇异值，$u_i$, $m_i$是对应的奇异向量。

<img src="../assets/image-20240508150256289.png" alt="image-20240508150256289" style="zoom:80%;" />



**GEAR Algorithm Overriew**

<img src="../assets/image-20240508150753550.png" alt="image-20240508150753550" style="zoom:80%;" />



**Hyper-param Choice**

+ outlier-reduced quantization

  在原文中sparsity ratio $s\in \{5\%,10\%\}$，compression ratio $r = \dfrac{1}{\frac{b}{16}+3s}$

+ low rank approximation

  但是在GEAR中sparsity ratio $s\in \{1\%,2\%\}$, 因为有$L$矩阵减少了误差。low-rank ratio $\rho \in \{2\%,5\%\}$, rank $L=\dfrac{1}{\frac{b}{16}+3s+\frac{n+d}{max(n,d)}\rho}$