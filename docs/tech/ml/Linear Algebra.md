# Linear Algebra

### 矩阵分解

#### QR(正交向量)分解

$A=QR$

对于一个列向量线性无关的，$m\times n $实矩阵$A$，可分解成$A=QR$，其中$Q$的列向量组是正交单位向量组，$R$为主对角元全为正数的上三角矩阵

#### 特征值分解

特征值：$Ax=\lambda x $，其中$A$是一个$n\times n$矩阵，$x$是矩阵$A$的特征值$\lambda$所对应的特征向量。

标准化：对于n个特征向量$\omega_1,...,\omega_n$，进行标准化(向量除以向量模长)后满足$||\omega_i||^2=1\text{ or }\omega_i^T\omega_i=1$，称这n个特征向量为标准正交基。此时$W$满足$W^T=W^{-1}$和$W^TW=I$。

对于一个n阶方阵$A$，n个特征值分别为$\lambda_1\leq \lambda_2\leq ...\leq \lambda_n$，以及对应的特征向量$\omega_1,...,\omega_n$，那么矩阵$A$就可以进行如下分解：

$$A=W\Sigma W^{-1}$$

其中$W$是由这n个特征向量组成的n阶方阵，$\Sigma$是这n个特征值维主对角线的n阶方阵

当把$W$的n个特征向量标准化后，特征分解可以写成$A=W\Sigma W^T$

#### SVD(奇异值)分解

$A=U\Sigma V^T$

对于一个$m\times n$实矩阵$A$，可分解成$A=U\Sigma V^T$，其中$U$是$m$级方阵，$\Sigma$是一个$m\times n$的矩阵，除了主对角上的元素以外全为0，主对角元$\sigma_1,...,\sigma_1$全为非负数称为奇异值，$V$是n级方阵，满足$U^TU=I, V^TV=I,\Sigma^T=\Sigma$

那么如何求解三个矩阵呢：

+ 对于$V$来说，利用方阵$(A^TA)v_i=\lambda_iv_i$，对其进行特征值分解，那么将$\lambda_1,...,\lambda_n$组成一个m阶方阵$V$，称为右奇异矩阵

+ 对于$U$来说，利用方阵$(AA^T)u_i=\lambda_iu_i$，对其进行特征值分解，那么将$\lambda_1,...,\lambda_n$组成一个n阶方阵$U$，称为左奇异矩阵

+ 对于$\Sigma$来说，由于除了主对角上的元素以外全为0，那么可以根据如下推导式求出每个奇异值$\sigma$，$A=U\Sigma V^T \Rightarrow AV = U\Sigma \Rightarrow Av_i=\sigma_iu_i \Rightarrow \sigma_i=Av_i/u_i$

  此外，$A^TA$特征值矩阵等于奇异值矩阵的平方，所以也可以通过如下求出$\sigma_i$

  $\sigma_i=\sqrt{\lambda_i}$

SVD的特性：

+ 奇异值矩阵中奇异值按照从大到小排列，并且下降很快在，很多情况下，前10%甚至1%的奇异值的和就占了全部的奇异值之和的99%以上的比例。

  因此可以对$\Sigma$矩阵进行近似，即用最大的k个奇异值和对应的左右奇异向量来近似矩阵，可以用来舍弃不重要的特征值

  $A_{m\times n}=U_{m\times m}\Sigma_ {m\times n}V^T_{n\times n} \approx  U_{m\times m}\Sigma_ {k\times k}V^T_{n\times n}$

<img src="C:\Users\squarehuang\AppData\Roaming\Typora\typora-user-images\image-20240614111104372.png" alt="image-20240614111104372" style="zoom:67%;" />

+ 左奇异矩阵可以用于行数的压缩，右奇异矩阵可以用于列数即特征维度的压缩

#### LU(三角)分解

$A=LU$

对于一个可逆方阵$A$，如果它的所有顺序主子式都不为0，可以分解成一个上三角矩阵和一个下三角矩阵的乘积形式，即$A=LU$



### 拉格朗日乘子法

[reference:优化-拉格朗日乘子法](https://zhuanlan.zhihu.com/p/154517678)

[非线性优化中的 KKT 条件该如何理解](https://www.zhihu.com/question/23311674/answer/468804362)

+ 拉格朗日乘子法是是一种寻找多元函数**在一组约束下**的**极值**的方法
+ 通过引入拉格朗日乘子，可将有 𝑑 个变量与 𝑘 个约束条件的最优化问题转化为具有 𝑑+𝑘 个变量的无约束优化问题求解

**等式情况**

对于$f(x,y)$在约束条件$g(x,y)=0$下的极值，即$$minf(x) \quad s.t.g(x)=0$$

根据如下极值点条件，便可以得到拉格朗日函数$L(\lambda,x)=f(x)+\lambda g(x)$，等同于求该函数的极值点

$\begin{equation}\left\{\begin{array}{l} \nabla f(\textbf x) + \lambda\nabla g(\textbf x)=0 \\ g(\textbf x) =0 \end{array}\right.\end{equation}$

**不等式情况**

对于$f(x,y)$在约束条件$g(x,y)$下的极值，即$$minf(x) \quad s.t.g(x)\leq0$$

可以转化成在如下三个约束下最小化拉格朗日函数$L(\lambda,x)=f(x)+\lambda g(x)$，这个式子称为KKT条件

$\begin{equation}\left\{\begin{array}{l} g(x)\leq0 \\ \lambda \geq 0 \\ \lambda g(x) = 0\end{array}\right.\end{equation}$

**一般情况**

多元函数的梯度：给定$f(x,y,z)$，$\nabla f =\vec e_x\frac{\partial}{\partial x}+\vec e_y\frac{\partial}{\partial y}+\vec e_z\frac{\partial}{\partial z}$，其中$\vec e_i$均为各方向上的单位向量

对于KKT条件，给定优化问题，在若干等式和不等式约束条件下，以n维向量$x$维自变量最大化目标函数$f$

$$max_xf(x) \quad s.t. h_j(x)=0,j=1,\dots,m \text{\quad and \quad} g_i(x)\leq 0,i=1,\dots,n$$

那么根据KKT条件，极值点满足如下三个式子
$$
\nabla f(x^*)=\sum_j\lambda_j\nabla h_j(x^*)+\sum_i\mu_i\nabla g_i(x^*)
\\ \mu_i\geq 0
\\ \mu_ig_i(x^*)=0
$$
**直观理解**：

给定一个优化问题，我们把满足**所有约束条件的n维空间区域**称为可行域。从可行域中的每一个点 $x$朝某个方向 $v$出发走一点点，如果还在可行域中，或者偏离可行域的程度很小，准确地说，偏移量是行进距离的**高阶无穷小量**，那么我们就说$v$是一个可行方向。我们用 $F(x)$表示点$x$的所有可行方向的集合 。 

对于可行域中的一个极大值点$x^*$ ，它的可行方向集合为 $F(x^*)$，从 $x^*$朝 $F(x^*) $中的某个方向走一小步，那么落点仍然（近似）在可行域中。 $x^*$是局部最大值点就意味着在这些**可行方向上目标函数 $f(x)$不能增大**，从而我们得到下面这样一个结论：

 在极值点$x^*$ ，让目标函数增大的方向不能在$F(x^*)$中。