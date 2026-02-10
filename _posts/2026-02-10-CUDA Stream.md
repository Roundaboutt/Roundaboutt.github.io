---
layout: post
title: "CUDA Stream"
date: 2026-02-17
categories: code
author: "Layla"
---

# CUDA Stream

在解释什么是CUDA Stream之前，我们先来考虑一个单任务在执行的流程：

```text
时间轴：
───────────────────────────────────────────────────────────►
CPU:  [准备数据A]               [处理结果]        [空闲]
                  ↓                    ↑
GPU:          [传输A]  [计算A]  [传回A]    [空闲GPU]
              └────────串行执行────────┘
```

在这种模式下，所有的操作都排成一条直线。你会发现，在任何一个垂直的时间点上，只有一个硬件单元在干活。当 GPU 在算数时，PCIe 传输通道（H2D/D2H）完全在带薪休假；而当数据传输时，昂贵的计算核心又在摸鱼。

既然PCIe和计算核心是两个不同的硬件单元，那为什么不能以流水线的方式来异步执行呢？

```text
时间轴：
─────────────────────────────────────────────────►
Stream 0:  [传A0] [算A0] [回A0]
Stream 1:         [传A1] [算A1] [回A1]
Stream 2:                [传A2] [算A2] [回A2]
Stream 3:                       [传A3] [算A3] [回A3]
           └──────重叠！并发！──────┘
```

在这种方式下，PCIe和计算单元都在一刻不停地工作。



## 什么是Stream？

你可以把 GPU 想象成一个拥有成千上万个工人的庞大工厂，而 **CUDA Stream** 就是工厂里的一条条**独立的作业流水线**。

在默认情况下，如果你不特意去创建 stream ，所有的任务（无论是搬运数据还是启动计算核函数）都会被扔进一个默认的“0号 Stream ”里。这就好比整个工厂只有一条传送带，不管你是要把原材料运进来，还是要把零件组装起来，所有人都得在这条传送带上按顺序排队。

这就解释了为什么 Stream 的本质其实是一个**硬件指令队列**。当你把一个任务放入某个 Stream 时，你实际上是在说：“在这个 Stream 里，必须先完成 A 才能做 B。”但是——这才是重点——**不同 Stream 之间的任务是没有必然先后顺序的**。如果不特意去创建Stream，就会出现上面所说的第一种情况：数据传输与数据计算在串行执行，效率十分低下。



### 逻辑独立性与物理竞争

虽然在软件层面 Stream 是完全独立的，但在物理层面，它们还是在竞争同一块蛋糕。比如，所有的 Stream 都要共用 PCIe 总线的带宽，所有的核函数都要去抢那几个显存控制器和计算单元。

这就解释了为什么 Stream 的数量并不是越多越好。如果你开了 1000 个 Stream ，但 PCIe 带宽已经被前 5 个流填满了，剩下的 995 个 Stream 也只能排队等待带宽。当多个 Stream 的核函数同时运行时，如果它们加起来需要的寄存器或共享内存超过了 GPU 的上限，硬件也会根据优先级进行排队。

---

### Pinned Memory

这里我必须再强调一个容易被忽视的细节：如果你想让 Stream 发挥作用，Host 端的内存必须是 **Pinned Memory**。

什么是Pinned Memory？简单来说，**Pinned Memory**（锁页内存，也叫 Page-locked Memory）就是一块在物理内存中被“钉死”的区域。

在普通的 C++ 编程中，你用 `new` 或者 `malloc` 申请的内存叫做 **Pageable Memory**（可分页内存）。为了让有限的内存装下更多的程序，操作系统有个很聪明的招数：虚拟内存管理。如果你有一块内存暂时没用，操作系统会把它偷偷搬到硬盘的交换区（Swap Space），把宝贵的物理内存腾出来。当你下次再访问它时，系统再把它搬回来。

这意味着，普通内存的**物理地址**是会随时变化的。

这就解释了为什么 GPU 搬运数据时会遇到麻烦。GPU 搬运数据主要靠 **DMA**（直接内存访问）引擎，这个引擎很死板，它必须知道确切的物理地址才能干活。如果你在搬运过程中，操作系统突然把这块内存“分页”到了硬盘，或者换了个物理位置，DMA 就会抓瞎，甚至导致系统崩溃。

为了安全起见，当你使用普通的 `cudaMemcpy` 搬运 Pageable Memory 时，CUDA 驱动程序其实在后台偷偷做了两步工作：

1.  先申请一块临时的 Pinned Memory。
2.  把你的数据从普通内存拷贝到这块 Pinned Memory，然后再由 DMA 把它推向 GPU。

这就产生了一次额外的内存拷贝开销！而如果你直接申请 **Pinned Memory**（在 CUDA 里使用 `cudaHostAlloc`），你实际上是和操作系统签了一个协议：“这块地我要了，无论发生什么，你都不准把它挪走，也不准把它存到硬盘里。”

因为物理地址固定了，DMA 引擎可以直接从这块内存抓数据往 GPU 塞，跳过了中间商赚差价。这就是为什么 Pinned Memory 的传输带宽通常比普通内存高得多。



更核心的一点是，如果想实现 **Stream 异步重叠**，Pinned Memory 是强制性的前提。

由于 Pinned Memory 不需要 CPU 参与数据搬运的准备工作（因为物理地址已知且固定），CPU 只要下个令，DMA 就可以独立完成任务。这就释放了 CPU，让它能立刻去调度其他的 Stream。如果用的是普通内存，CPU 会被强行留下来配合驱动程序做同步锁定，异步 Stream 也就退化成了一个 Stream 串行执行。

不过，这么好的东西为什么不全部都用？

因为 Pinned Memory 是直接抢占物理内存份额的。它不能被交换到硬盘，意味着如果你申请了太多的 Pinned Memory，系统的可用内存就会急剧萎缩。所以，我们通常只对那些**频繁需要与 GPU 交换数据**的核心缓冲区使用 `cudaHostAlloc`。

```cpp
// 错误：异步传输用普通内存（不会报错，但不异步！）
float *h_data = (float*)malloc(size);
cudaMemcpyAsync(d_data, h_data, size, ..., stream);  // 实际是同步！

// 正确：异步传输用Pinned Memory
float *h_data;
cudaMallocHost(&h_data, size);  // Pinned Memory
cudaMemcpyAsync(d_data, h_data, size, ..., stream);  // 真正异步
```

---

## Stream 异步实现多batch矩阵乘法

假设我们要计算100个矩阵A和同一个矩阵B的计算结果，也就是100个矩阵C。在同步执行中，我们需要将这100次矩阵乘法串行执行。

下面我们来看看如何用 Stream 异步实现。



首先是kernel：

```cpp
__global__ void gemm_naive(float* A, float* B, float* C, int M, int N, int K) {
    const int idx = threadIdx.x + blockDim.x * blockIdx.x;
    const int idy = threadIdx.y + blockDim.y * blockIdx.y;

    if (idy < M && idx < N) {
        float temp = 0;
        for (int k = 0; k < K; k++) {
            // 核心逻辑：A的行乘以B的列
            temp += A[idy * K + k] * B[k * N + idx];
        }
        C[idy * N + idx] = temp;
    }
}
```

这里就不多讲了。



批次大小，以及三个矩阵的维度：

```cpp
const int num_batch = 100;
const int M = 1024, N = 1024, K = 512;
const size_t Asize = M * K, Bsize = K * N, Csize = M * N;
```



使用 Pinned Memory 分配 Host 端内存。

注意这里的 h_A 和 h_C，实际上是一个二维数组\[num_batch][size]压成了一维。为什么写成二维数组然后循环分配内存呢？因为调用一次malloc开销太大了！不如提前把所有的都分配好。

```cpp
float *h_A, *h_B, *h_C;
cudaMallocHost((void**)&h_A, num_batch * Asize * sizeof(float));
cudaMallocHost((void**)&h_B, Bsize * sizeof(float)); // B 是共享的
cudaMallocHost((void**)&h_C, num_batch * Csize * sizeof(float));
```



初始化数据：

```cpp
init_data(h_A, num_batch * Asize);
init_data(h_B, Bsize);
```



 预先分配 Device 端内存，避免在循环中 malloc：

```cpp
float *d_A, *d_B, *d_C;
cudaMalloc((void**)&d_A, num_batch * Asize * sizeof(float));
cudaMalloc((void**)&d_B, Bsize * sizeof(float));
cudaMalloc((void**)&d_C, num_batch * Csize * sizeof(float));
// 先把共享的 B 矩阵传过去
cudaMemcpy(d_B, h_B, Bsize * sizeof(float), cudaMemcpyHostToDevice);
```



创建 Stream Pool：

```cpp
const int pool_size = 8;
cudaStream_t streams[pool_size];
for (int i = 0; i < pool_size; i++) cudaStreamCreate(&streams[i]);
```



执行异步流水线：

```cpp
dim3 blocksize(32, 32);
dim3 gridsize((N + blocksize.x - 1) / blocksize.x, (M + blocksize.y - 1) / blocksize.y);

for (int i = 0; i < num_batch; i++) {
    int s_idx = i % pool_size;	// 每个batch 循环使用 stream pool

    float* cur_h_A = h_A + i * Asize;
    float* cur_d_A = d_A + i * Asize;
    float* cur_d_C = d_C + i * Csize;
    float* cur_h_C = h_C + i * Csize;
	// 异步传输数据（使用PCIe总线 H2D）
    cudaMemcpyAsync(cur_d_A, cur_h_A, Asize * sizeof(float), cudaMemcpyHostToDevice, streams[s_idx]);
	// 计算
    gemm_naive<<<gridsize, blocksize, 0, streams[s_idx]>>>(cur_d_A, d_B, cur_d_C, M, N, K);
	// 异步传输数据（使用PCIe总线 D2H)
    cudaMemcpyAsync(cur_h_C, cur_d_C, Csize * sizeof(float), cudaMemcpyDeviceToHost, streams[s_idx]);
}
```



最后是验证结果与释放内存：

```cpp
// 等待所有任务完成并清理
cudaDeviceSynchronize();

std::cout << "正在验证第 0 个 Batch 的计算结果..." << std::endl;
bool is_correct = verify_result(h_A, h_B, h_C, M, N, K);

if (is_correct) {
    std::cout << "验证通过！" << std::endl;
} else {
    std::cout << "错误！" << std::endl;
}

for (int i = 0; i < pool_size; i++) cudaStreamDestroy(streams[i]);
cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
cudaFreeHost(h_A); cudaFreeHost(h_B); cudaFreeHost(h_C);
```



完整代码：

```cpp
#include <cuda_runtime.h>
#include <iostream>
#include <vector>

__global__ void gemm_naive(float* A, float* B, float* C, int M, int N, int K) {
    const int idx = threadIdx.x + blockDim.x * blockIdx.x;
    const int idy = threadIdx.y + blockDim.y * blockIdx.y;

    if (idy < M && idx < N) {
        float temp = 0;
        for (int k = 0; k < K; k++) {
            temp += A[idy * K + k] * B[k * N + idx];
        }
        C[idy * N + idx] = temp;
    }
}

void init_data(float* data, size_t size) {
    for (size_t i = 0; i < size; ++i) {
        data[i] = static_cast<float>(rand() % 5);
    }
}

bool verify_result(float* host_A, float* host_B, float* gpu_res, int M, int N, int K) {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            float cpu_sum = 0;
            for (int k = 0; k < K; ++k) {
                cpu_sum += host_A[i * K + k] * host_B[k * N + j];
            }
            if (std::abs(cpu_sum - gpu_res[i * N + j]) > 1e-5) {
                std::cout << "验证失败! 位置: [" << i << "," << j << "] "
                          << "CPU: " << cpu_sum << " GPU: " << gpu_res[i * N + j] << std::endl;
                return false;
            }
        }
    }
    return true;
}

int main() {
    const int num_batch = 100;
    const int M = 1024, N = 1024, K = 512;
    const size_t Asize = M * K, Bsize = K * N, Csize = M * N;

    // 使用 Pinned Memory 分配 Host 端内存
    float *h_A, *h_B, *h_C;
    cudaMallocHost((void**)&h_A, num_batch * Asize * sizeof(float));
    cudaMallocHost((void**)&h_B, Bsize * sizeof(float)); // B 是共享的
    cudaMallocHost((void**)&h_C, num_batch * Csize * sizeof(float));

    // 初始化数据
    init_data(h_A, num_batch * Asize);
    init_data(h_B, Bsize);

    // 2. 预先分配 Device 端内存，避免在循环中 malloc
    float *d_A, *d_B, *d_C;
    cudaMalloc((void**)&d_A, num_batch * Asize * sizeof(float));
    cudaMalloc((void**)&d_B, Bsize * sizeof(float));
    cudaMalloc((void**)&d_C, num_batch * Csize * sizeof(float));

    // 先把共享的 B 矩阵传过去
    cudaMemcpy(d_B, h_B, Bsize * sizeof(float), cudaMemcpyHostToDevice);

    const int pool_size = 8;
    cudaStream_t streams[pool_size];
    for (int i = 0; i < pool_size; i++) cudaStreamCreate(&streams[i]);

    // 执行异步流水线
    dim3 blocksize(32, 32);
    dim3 gridsize((N + blocksize.x - 1) / blocksize.x, (M + blocksize.y - 1) / blocksize.y);

    for (int i = 0; i < num_batch; i++) {
        int s_idx = i % pool_size;
        
        float* cur_h_A = h_A + i * Asize;
        float* cur_d_A = d_A + i * Asize;
        float* cur_d_C = d_C + i * Csize;
        float* cur_h_C = h_C + i * Csize;

        cudaMemcpyAsync(cur_d_A, cur_h_A, Asize * sizeof(float), cudaMemcpyHostToDevice, streams[s_idx]);
        
        gemm_naive<<<gridsize, blocksize, 0, streams[s_idx]>>>(cur_d_A, d_B, cur_d_C, M, N, K);
        
        cudaMemcpyAsync(cur_h_C, cur_d_C, Csize * sizeof(float), cudaMemcpyDeviceToHost, streams[s_idx]);
    }

    // 等待所有任务完成并清理
    cudaDeviceSynchronize();
    
    std::cout << "正在验证第 0 个 Batch 的计算结果..." << std::endl;
    bool is_correct = verify_result(h_A, h_B, h_C, M, N, K);

    if (is_correct) {
        std::cout << "验证通过！" << std::endl;
    } else {
        std::cout << "错误！" << std::endl;
    }

    for (int i = 0; i < pool_size; i++) cudaStreamDestroy(streams[i]);
    cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
    cudaFreeHost(h_A); cudaFreeHost(h_B); cudaFreeHost(h_C);
}
```



















