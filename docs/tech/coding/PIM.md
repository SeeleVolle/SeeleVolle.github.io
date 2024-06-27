# PIM Learning

### Coding on UPMEM

[Standard library functions — UPMEM DPU SDK 2024.1.0 Documentation](https://sdk.upmem.com/2024.1.0/04_Stdlib.html)

#### DEBUG

一些dpu-lldb常见操作：

```shell
dpu-lldb
process launch --stop-at-entry
process continue
frame variable/x checksum
target variable/x checksum
breakpoint set --source-pattern-regexp "return"
parray/x 20 &input[0]
memory read
memory write -i sample.bin '&buffer[0]'
```

#### 线程管理和同步

在UPMEM的DPU中，将线程抽象为Tasklet，提供了Mutexes, Semaphore, Barrier, Handshake的四种进程同步方法

**运行参数**

- `NR_TASKLETS` 用于定义tasklet的数量。
- `STACK_SIZE_DEFAULT` 用于定义所有未指定堆栈的tasklet的堆栈大小。
- `STACK_SIZE_TASKLET_<X>` 用于定义特定tasklet 的堆栈大小。

```shell
dpu-upmem-dpurte-clang -DNR_TASKLETS=3 -DSTACK_SIZE_DEFAULT=256 -DSTACK_SIZE_TASKLET_1=2048 -O2 -o tasklet_stack_check tasklet_stack_check.c
```

**Mutexes**

三个函数，初始化mutex，获取mutex，解锁mutex

```cpp
MUTEX_INIT(my_mutex);
mutex_lock(my_mutex);
mutex_unlock(my_mutex);
```

**Semaphore**

一般仅用于需要计数器同步的情况，`sem_take` 和 `sem_give` 分别减少和增加信号量计数器。如果计数器为零或小于零， `sem_take `会挂起调用的tasklet 执行，直到另一个tasklet 调用 `sem_give`

```cpp
SEMAPHORE_INIT(my_semaphore, 0);
sem_give(&my_semaphore);
sem_take(&my_semaphore)
```

**Barrier**

可以确保特定数量的tasklet在继续运行之前运行到同一起点上，先初始化barrier等待的tasklet数目，然后调用就可以wait

```cpp
BARRIER_INIT(my_barrier, NR_TASKLETS_NUMBER);
barrier_wait(&my_barrier);
```

**Handshake**

Handshake指的是可以实现在其中一个Tasklet完成特定任务后启动另一个Tasklet，通过wait_for和notify实现，其中wait_for的参数是tasklet通过`me()`方法获取的sysname_t

```cpp
handshake_wait_for(sysname_t notifier) 
handshake_notify(void)
```

#### **内存管理**

**内存模型**

在UPMEM的DPU中，有两种用途的Memory，分别是**WRAM**和**MRAM**，其中WRAM是程序执行时的内存，存放了stack, 全局变量和heap和手动分配的shared memory等资源，而MRAM可以被理解为“external peripheral”，访存速度较慢。

**WRAM Heap Allocation**

有三种allocator可以在WRAM上分配，分别是incremental allocator, fixed-size block allocator, buddy allocator.

+ Incremental allocator：使用这种allocator时，有几点需要注意
  + Runtime Library对于WRAM的组织方式是，除了program运行时所需的内存，剩余的所有内存都作为"free area"
  + 没有free方法，一个task动态申请一个Buffer后，这个Buffer会一直保留该task的property直到程序结束
  + `mem_reset()`方法用于clean-up heap，例如当一个DPU被程序多次启动时

```cpp
mem_alloc(size_t size)
mem_reset()
```

+ Fixed-Size block Allocator: 用于用户分配固定块大小的内存
  + 需要使用`fsb_allocator_t`，通过`fsb_alloc()`来分配空间，并通过`fsb_get()`来获取一个allocated block

```cpp
fsb_alloc(unsigned int block_size, unsigned int nb_of_blocks)
fsb_get(fsb_allocator_t allocator)
fsb_free(fsb_allocator_t allocator, void* ptr)    
```

+ Buddy Allocator: 即OS中提到的buddy allocation method
  + 分配的内存大小不能超过4096B，最小是32B，自动和DMA-transfer-size对齐

```cpp
buddy_init(unsigned int buffer_size)
buddy_alloc(unsigned int buffer_size))
buddy_free(char * memory)
```

**MRAM Management**

三种变量声明，`__mram, __mram_noinit, __mram_ptr`，前者将变量声明到MRAM上，中间代表该变量没有初始值，后者代表一个指针指向MRAM上的变量或者声明一个extern的MRAM variable.

在一般情况下，对于一个在MRAM中的8B变量，会由WRAM中的pre-defined cache来处理。**Implicit write access of MRAM variables**会经历如下三个过程：从WRAM cache中读8B，修改其中的x bytes，再把8B写回到cache中，并且8B以下的修改是非多线程安全的。

**Direct access to the MRAM**

主要由两个函数来操作：

+ 需要保证source和target的address都是8 bytes对齐的，并且transfer size必须是8的倍数，并且不能超过2048

```cpp
mram_write(const void *from, __mram_ptr void *to, unsigned int nb_of_bytes)
mram_read(const __mram_ptr void *from, void *to, unsigned int nb_of_bytes)
```

#### Exception

一个DPU Program可能因为三种fault停止，只能通过`dpu-lldb` 来打印stop reason，host API只能看到发生了错误

+ Memory fault

+ DMA fault

+ Runtime Library fault

#### Controlling the execuation of DPUs

**Allocation**

`dpu_alloc`: 分配指定数量的GPU

`dpu_free`: 释放一组由`dpu_alloc`分配的DPU

`dpu_get_nr_ranks`: 返回一个DPU set中的rank数量

`dpu_get_nr_dpus`: 返回一个DPU set中的DPU数量

**Loading Programs**

`dpu_load`：该函数输入一个二进制文件路径，加载二进制文件到指定的DPU上，并存储程序信息到给定的指针上。加载后，该程序持久保存在DPU内存中，可以由应用程序多次启动

`dpu_load_from_memory`：该函数加载一个在内存中的程序

**Executing Programs**

`dpu_launch`：该函数启动给定集合的所有DPU，某些资源会在启动前重置

+ `DPU_SYNCHRONOUS`：暂停应用程序，直到请求的 DPU 完成执行（或遇到错误）
+ `DPU_ASYNCHRONOUS` 立即将DPU的控制权交还给应用程序，应用程序将负责通过 `dpu_status` 或 `dpu_sync` 检查DPU的状态

#### Communication with host applications

**Memory Interface**

使用这些内存传输函数存在一些限制：

+ IRAM和MRAM地址要求8B对齐，WRAM地址要求4B对齐
+ symbol name是DPU code中的WRAM或者MRAM变量名组成
+ 与 DPU WRAM 的通信比与 MRAM 之间的复制操作更慢。此外，与 MRAM 相比，WRAM 是更小的存储器。因此，DPU WRAM 应该用于共享少量数据，大数据应放到MRAM中
+ 每个DPU只能访问自己的MRAM中的数据，所以DPU执行应尽可能独立于外部数据

`dpu_copy_from(struct dpu_set_t set, const char *symbol_name, uint32_t symbol_offset, void *dst, size_t length)`: 从DPU上copy到buffer中

`dpu_broadcast_to(struct dpu_set_t set, const char *symbol_name, uint32_t symbol_offset, const void *src, size_t length, dpu_xfer_flags_t flags)`：将一个buffer广播到所有DPU中

`dpu_broadcast_to(struct dpu_set_t set, const char *symbol_name, uint32_t symbol_offset, const void *src, size_t length, dpu_xfer_flags_t flags)`：在一次传输中将不同的buffer传输到一组DPU中

**Rank Transfer Interface**

`dpu_prepare_xfer`：将buffer属性赋给一组DPU，可以在`dpu_push_xfer`方法中使用

`dpu_push_xfer`：使用之前定义的缓冲区，按照给定方向，DPU symbol name，DPU symbol length进行传输

