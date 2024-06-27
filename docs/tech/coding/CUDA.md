## Cuda Learning

### 内存架构模型

[GPU内存(显存)的理解与基本使用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/462191421)

**Zero copy的使用**

+ GPU直接从系统内存读取数据
+ 适用于只需要一次读取或者写入的数据操作，要频繁读写的数据，不建议用zero copy。
+ Pinned memory: 数据会在内存中锁住，不会被换到硬盘中

```c++
float *a_h, *a_map; // 定义两个指针：a_h 内存原指针，a_map映射指针 
...
cudaGetDeviceProperties(&prop, 0);       // 获取GPU的特性，看是否支持地址映射 
if (!prop.canMapHostMemory) 
    exit(0);
cudaSetDeviceFlags(cudaDeviceMapHost);    // 设置设备属性，打开地址映射
cudaHostAlloc(&a_h, nBytes, cudaHostAllocMapped);  // 开辟pinned memory
cudaHostGetDevicePointer(&a_map, a_h, 0);    // 地址映射 a_h ->  a_map.
kernel<<<gridSize, blockSize>>>(a_map);   
```

**设备内Async 拷贝**

+ 用于全局内存和共享内存之间传递数据，与`cudaMemcpyAsync`区别开
+ 数据由L2直接传递到SMEM，而不需要经过L1和RF的中转

```cpp
#include <cooperative_groups.h>
#include <cooperative_groups/memcpy_async.h>

namespace cg = cooperative_groups;

__global__ void kernel(int* global_data) {
    cg::thread_block tb = cg::this_thread_block();
    const size_t elementsPerThreadBlock = 16 * 1024;
    const size_t elementsInShared = 128;
    __shared__ int local_smem[elementsInShared];

    size_t copy_count;
    size_t index = 0;
    while (index < elementsPerThreadBlock) {
        cg::memcpy_async(tb, local_smem, elementsInShared, global_data + index, elementsPerThreadBlock - index);
        copy_count = min(elementsInShared, elementsPerThreadBlock - index);
        cg::wait(tb);
        // Work with local_smem
        index += copy_count;
    }
}
```

**设备间数据传输(NVLINK)**

```cpp
float* p0, *p1;
cudaSetDevice(0);                   // 将GPU0设置为当前设备
size_t size = N * sizeof(float);    // size设置为N个 float
cudaMalloc(&p0, size);              // GPU0开辟内存
cudaSetDevice(1);                   // 将GPU1设置为当前设备
cudaMalloc(&p1, size);              // GPU0开辟内存
cudaMemcpyPeer(p1, 1, p0, 0, size); // Copy p0 to p1
```

**内存种类**

+ Global mmeory：Gmem可以被设备内的所有线程访问，off chip，

```cpp
cudaMalloc() //分配设备内存
cudaMallocPitch() //分配设备上的二维数据并进行对齐
cudaMallocManaged() //分配内存并交给UA管理
cudaMallocHost() //在主机上分配pinned-memory
cudaMallocAsync() //Stream-ordered操作来执行内存分配和释放，
```

[CUDA编程入门之 Stream-Ordered Memory Allocator(1) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/602591368)

二维矩阵的访问：

```cpp
for (int i=0;i<Height;i++){
    float *row = (float *)((char*)N+i*Pitch);
    for (int j=0;j<Width;j++){
        row[j]++;
    }
}
```

__threadfence()：保证调用该函数的线程，在该语句前对全局存储器或者共享存储器的访问已经全部完成，执行结果对grid中的所有线程可见。

+ L1/L2 cache：L2可以被所有SM访问，L1可以被当前SM内访问
+ Local memory：线程独享，off chip，当寄存器不足时添加
+ Register：on chip，线程独享，on chip
+ Shared memory: block内共享，用于缓存反复读写的数据，与L1的位置和速度相似, onchip
+ constanct memory: 通过cache的多副本，减少多线程并行访问常量的延时，host allocation
+ texture memory:off chip，用于图形化数据的存储，host allocation