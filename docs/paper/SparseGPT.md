# SparseGPT:Massive Language Models Can Be Accurately Pruned in One-Shot

核心：通过近似算法解决了Reconstruction的时间复杂度过高的问题，解决了Hessians矩阵在不同行之间不同所引入的求逆复杂度。

Post-Training Pruning: PTP问题就是对于一个预训练好的模型，在给定的Calibration data下寻找输出变化最小的稀疏模型。

Layer-Wise Pruning: 对于PTP来说，通常分解为分层的子问题。对于每个layer $l$，就是要找到一个具有目标稀疏度的最优掩码$M_l$和更新后的权重$W_l$让这个式子代表的loss最小：(即每一层的输出相较于原来的输出)

<img src="../assets/image-20240429144012801.png" alt="image-20240429144012801" style="zoom:67%;" />

Mask Selection & Weight Reconstruction：通常情况下，解决Pruning问题分为两个步骤，第一个步骤是根据显著性标准找到一个Mask，第二个步骤是在Mask不变的情况下，优化剩下未修剪的权重。

**Fast Approximate Reconstruction**

对于固定的pruning mask M，可以通过求解对应矩阵每行$w^i$的稀疏重构问题（一个线性平方误差问题？）来精确计算mask中所有权重的最优值，即用标准的线性回归公式：

$w^i_{M_i}=(X_{M_i}X^T_{M_i})^{-1}X_{M_i}(w_{M_i}X^T_{M_i})$，其中$X_{M_i}$代表的是subset of input features中没被剪掉的部分，而$w_{M_i}$是它们对应的weight。

>  由于求解$H_{M_i}=X_{M_i}X^T_{M_i}$的逆矩阵的时间复杂度是$O(d_{row}*d_{col}^3)(d_{row}\approx d_{col}\approx d_{hidden})$，几乎是不可接受的，因为每一行的Pruning mask不同，所以求逆需要每行对一个$d_{col}\times d_{col}$的矩阵单独求逆。如果行掩码都相同，那么只计算一个单独的共享逆即可。

OBS方法：一个迭代式的行权重重建方法。假设$w_m$是对应掩码确定后的权重，那么$w+\delta_m$就是最优权重重建。同理，对于给定掩码$M$的最优稀疏重构$w^{(M)}$，同样运用OBS可以找到最优掩码重构$M'=M-{m}$。

<img src="../assets/image-20240429151612310.png" alt="image-20240429151612310" style="zoom:67%;" />

通过使用OBS，可以Iterative(逐行地)进行权重修剪，从而和之前一样对于固定的掩码M，可以同样的重建得到相同的最优解。

OPU局部更新：考虑在OBS中只更新可用参数的一个子集$U\in M$。基于$|U|<|M|$时，求逆$H_U$会比$H_M$更快的 事实。其实关键在于定义了这样一个

Hessian同步：基于OBS和OPU，SparseGPT实现了Hession的同步。没太看懂.....从横着剪变成了竖着剪？

> 递归定义子集序：$U_{j+1}=U_j-{j}, with\ U_1=\{1,\dots,d_{col}\}$。这些子集的Hessian矩阵的逆$(H_{U_j})^{-1}=((XX^T)_{U_j})^{-1}$可以在W的各个行间共享（因为输入都是一样的）, 记$B=(H_{U_j})^{-1}$, 可以得到更新后的$(H_{U_{j+1}})^{-1}=(B-\dfrac{1}{[B]_{11}}\cdot B_{:,1}B_{1,:})_{2:,2:}$，其中$(H_{U_1})^{-1}=H^{-1}$。所以可以在$O(d_{col}^3)$内得到整个矩阵的逆Hessians矩阵。

如图所示：对于一个固定的mask M，修剪权重矩阵$W$时，逐步对每列进行修剪。按顺序迭代 $U_j$和对应的逆Hessian矩阵，并对所有的行 $i$ 剪去满足的$j∉M$的$w_j$， 同时对对应行剩下的weight进行更新。如图所示：

<img src="../assets/image-20240429154921948.png" alt="image-20240429154921948" style="zoom: 80%;" />

**Adaptive Mask Selection**

解决一个问题：如何获得$w$%的稀疏度，如果是在列修剪时选择$w$%的权重进行剪除，那么得到的权重矩阵不能在列之间非均匀分布（增加了额外的约束导致剪枝性能下降...）。SparseGPT通过迭代分块的方法来实现Adaptive Mask Selection，即对于$B_s$列作为一块，保证每块内的稀疏度。

每次选择$B_s$列的掩码执行权重更新，然后再选择下一个块的掩码。

<img src="../assets/image-20240429160258034.png" alt="image-20240429160258034" style="zoom:80%;" />

完整算法伪代码如下：

<img src="../assets/image-20240429155723087.png" alt="image-20240429155723087" style="zoom:80%;" />
