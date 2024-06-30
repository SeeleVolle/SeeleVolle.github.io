# 稀疏矩阵

### Sparse Matrix存取格式

在稀疏矩阵的存储中，由于矩阵的大部分值都是0，因此直接存储矩阵需要浪费大量空间，在计算时也会浪费大量时间。为此，通常由以下几种稀疏矩阵存储格式：

+ **COO**: 坐标存储，格式是(row indices, col indices,values)，通过行、列、值三个参数来表示稀疏矩阵
+ **F-COO**:
+ **Hi-COO**:
+ **CSR**: 行压缩存储，格式是(row offsets, column indices, values)，压缩了COO方法中的行数组，表示当前行第一个元素在顺序数组中的下标
+ **CSC**：列压缩存储，格式是(row indices, col offsets, values)，压缩了COO方法中的列数组，表示当前列第一个元素在顺序数组中的下标


### SpGEMM

Waiting for Completion...

### SpGETT

参见SpGETT.md