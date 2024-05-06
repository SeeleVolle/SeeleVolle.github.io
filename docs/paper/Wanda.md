# Wanda

剪枝算法核心：

+ 对于每个weight，使用它的magnitude和对应input activations的norm的乘积来进行评估
+ On a **per-output** basis instead of across the whole layer(weight importance score are compared within **each row in W**)

Magnitude Pruning: 将权重低于某个阈值的全部剪枝，之后进行fine-tuned直到准确率到达预期。

$$M_{ij}= Top_v(S)_{ij}= \begin{cases} 1& \text{if\ Sij in top v\% }\\ 0& \text{otherwise} \end{cases}$$，其中$S_{ij}$表示重要性分数。对于ReLU来说$Y=max(0,XW)$，结果就变成了$Y=max(0,X(W\odot M))$，$\odot$表示Hadamard积，黑色代表1。

<img src="../assets/image-20240429115511549.png" alt="image-20240429115511549" style="zoom:67%;" />

Wanda的**核心过程**Overview如下图所示：

+ 两种Weight Update
  + Sequential：先计算出完整的pruned mask，再对权重进行更新
  + Iterative: SparseGPT所采用的方法， 在每一个Layer内部迭代的进行剪枝和权重更新，SparseGPT将128个Input channel作为一个comparison group来进行更新。
+ Weight Update对Wanda的优化效果可以忽略（It offers **little or negligible** improvement to Wanda.）

<img src="../assets/image-20240429123533893.png" alt="image-20240429115511549" style="zoom:67%;" />

Pruning Metric: 

假设一个线性层的$W\in R^{C_{out}\times C_{in}}$, Input activations $X\in R^{NL\times C_{in}}$，对于某个weight$W_{ij}$，Score可以计算如下：$S_{ij}=|W_{ij}|\cdot||X_j||^2$，前者代表绝对值，后者是$j$th feature的L2范数(即第j行)

Comparison Group:

如上文所说，与传统的layer-wise和whole-network不同，Wanda中的Comparison group是output neuron，也就是output矩阵的每一行。用公式表示就是$For\ W_{ij},\ G_{ij}=\{W_{uv}|u=i\}$


