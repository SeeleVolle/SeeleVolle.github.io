# 内存硬件基础知识

[参考blog](http://www.biscuitos.cn/blog/Memory-Hardware)

## 存储器子系统

### 内存颗粒层次结构

<img src="../assets/Memory_hardware/Memorysubsystem.png" alt="Memorysubsystem.png" style="zoom:50%;" />

内存控制器到内存颗粒的内部逻辑，从大到小可以分为：
`Channel -> DIMM -> RANK -> CHIP -> BANK -> Row/Column`

+ **Memory Channel**: 内存通道(Channel) 是内存控制器和内存之间通信的总线，其中一端连接内存控制器，另一端连接DIMM插槽。
  <img src="../assets/Memory_hardware/Channel.png" alt="Channel.png" style="zoom:67%;" />
  在服务器上外观看来表现为紧紧相邻的一系列 DIMM 插槽，其中相邻两个 DIMM 插槽属于同一个内存 Channel，通常有单通道，双通道，四通道等。  
  如果要提高内存的访问量，那么可以同时使用多个内存控制器访问内存，因此可以将内存条插入到不同的 Channel，这样可以提供内存总体的访问速度(多通道速率 = 单通道速率 x 通道数)
  + 单通道模式：指的是提供单通道的带宽运算，在只安装一个 DIMM 或者安装的多个 DIMM 内存容量不同时启用，并且会采用通道慢的内存条
  + 双通道模式：当两个 DIMM 容量相等时，系统可以启用双通道来提高内存的吞吐量，默认会采用较慢的内存条。如图C所示，此时只要A1和B1, A2和B2的容量相同即可，无需全部相同。
  
+ **DIMM(Double-Inline Memory Module)**: DIMM 也就是我们俗称的内存条，实际上也就是一块焊有多个内存芯片的电路板。  
  <img src="../assets/Memory_hardware/DIMM.png" alt="DIMM.png" style="zoom:67%;" />
  DIMM 中的 double 指的是相较于从前的 SIMM 来说，它可以单个周期内读取 8 个字节，即位宽是 64 位。如今常见的 DIMM 类型如下：
  + RDIMM: Registered DIMM，主要用于服务器上，为了稳定性有 ECC 和无 ECC 两种，一般是自带 ECC 的
  + UDIMM: UnBuffered DIMM，主要用于个人 PC 上，同样有 ECC 和无 ECC 两种，一般不带 ECC
  + SO-DIMM: Small-Outline DIMM，主要用于笔记本，也分 ECC 和无 ECC 两种.
   
+ **Memory Rank/Chip**: Chip 是实际存储数据的地方，每个 DIMM 上会有多个类似的 Chip，单个 Chip 的位宽一般是4/8/16/32 bit，所以会将多个 Chip 并联起来组成 64 bit的位宽集合，这个集合就称为是 RANK.
 <img src="../assets/Memory_hardware/Chip.png" alt="Chip.png" style="zoom:67%;" />
 对于某些内存条来说，可能正反都有很多 Chips，那么正反两面每一个面的集合都叫做 RANK，即有两个 RANK

+ **Memory Bank/Cell**: Chip 内部包含了许多 Bank, 每一个 Bank 实际上是一个存储矩阵(Memory Array)，这些 Bank 共享 Chip 内的 BA(Bank Address)线，而在 Bank 内部通过行列划分为一个个独立的存储单元，称为 Cell，可以保存 1 bit 的数据，Cell 由 Row Decoder/Column Decoder 寻址。每组 Bank 的下方会有一个 Row Buffer(Sense amplifer) 来将读出的 Row 缓存
 <img src="../assets/Memory_hardware/Bank.png" alt="Bank.png" style="zoom:67%;" />

  内存控制器可以对同一个 Rank 内的所有 Chip 同时进行读写，同一个 Rank 内的所有 Chip 共享同样的控制信号，多个 Rank 可以共享同一根 Addr/Command 信号线，利用 CS 线来选择使用哪组 Rank. 


