---
layout: post
title:  "swizzling避免bank conflict"
date:   2026-02-9
categories: code
---

# swizzling避免bank conflict

## bank conflict产生的原因

快速回顾一下：bank conflict 发生在shared memory中，原因是一个shared memory分为32个bank，当一个warp中的多个线程同时访问一个bank中的不同地址时就会发生bank conflict，从而使内存访问串行化，大大降低了访存效率。

没有swizzling的原始版本：

```cpp
// kernel:
#define TILE_DIM 32
__global__ void transpose_v1(float* output, float* input, int nx, int ny) {
    __shared__ float smem[TILE_DIM][TILE_DIM];

    const int tx = threadIdx.x;
    const int ty = threadIdx.y;

    int id_x = blockIdx.x * TILE_DIM + tx;
    int id_y = blockIdx.y * TILE_DIM + ty;

    if (id_x < nx && id_y < ny) {
        smem[ty][tx] = input[id_y * nx + id_x];
    }

    __syncthreads();

    int trans_x = blockIdx.y * TILE_DIM + tx;
    int trans_y = blockIdx.x * TILE_DIM + ty;

    if (trans_x < ny && trans_y < nx)
    {
        output[trans_y * ny + trans_x] = smem[tx][ty];
    }
}

```

```cpp
void transpose_gpu(float* h_output, float* h_input, int nx, int ny)
{
    float *d_input, *d_output;
    size_t size = nx * ny * sizeof(float);

    cudaMalloc(&d_input, size);
    cudaMalloc(&d_output, size);

    cudaMemcpy(d_input, h_input, size, cudaMemcpyHostToDevice);

    dim3 block(32, 32);
    dim3 grid((nx + block.x - 1) / block.x, (ny + block.y - 1) / block.y);
    
    transpose_v1<<<grid, block>>>(d_output, d_input, nx, ny);

    cudaMemcpy(h_output, d_output, size, cudaMemcpyDeviceToHost);

    cudaFree(d_input);
    cudaFree(d_output);
}
```

在这个例子中，一个block中的所有thread协作处理一个tile（shared memory中的小块），所以blockDim必须与TILE_DIM相同，恰好都是(32, 32)。而每个bank的宽度是4B，一共32个bank，所以smem数组的每一列的元素都恰好位于同一个bank内。

现在问题就很明显了。

```cpp
if (trans_x < ny && trans_y < nx)
{
    output[trans_y * ny + trans_x] = smem[tx][ty];
}
```

在读取smem时，同一个warp从smem中取数据时（一个warp中的线程ty相同，tx不同），对应的都是smem中的同一列，也就是同一个bank！这就造成了32路bank conflict。

在ncu中也可以证实这一点：

![65f329cbf06c011c07cbcdc4c73d3def](\images\swizzling避免bank_conflict\1.png)

---

## swizzling如何解决bank conflict

```cpp
// swizzling kernel：
#define TILE_DIM 32
__global__ void transpose_v1(float* output, float* input, int nx, int ny) {
    __shared__ float smem[TILE_DIM][TILE_DIM];

    const int tx = threadIdx.x;
    const int ty = threadIdx.y;

    int id_x = blockIdx.x * TILE_DIM + tx;
    int id_y = blockIdx.y * TILE_DIM + ty;

    if (id_x < nx && id_y < ny) {
        smem[ty][tx ^ ty] = input[id_y * nx + id_x];
    }

    __syncthreads();

    int trans_x = blockIdx.y * TILE_DIM + tx;
    int trans_y = blockIdx.x * TILE_DIM + ty;

    if (trans_x < ny && trans_y < nx)
    {
        output[trans_y * ny + trans_x] = smem[tx][ty ^ tx];
    }
}
```

为什么一个简简单单的异或操作就能避免bank conflict呢？

<img src="\images\swizzling避免bank_conflict\2.png" alt="f6ad220af10eadd3a8c17fc0167829b2" style="zoom:50%;" />

在上图中，大的方格就是smem，小方格就是一个元素（float），其中的数字就代表它所在的bank编号。可以看出，在原始状态下，每一列都处在同一个bank；经过XOR swizzling之后，每一列元素所在的bank都互不相同了。从上图可知，swizzling的本质就在于**改变逻辑坐标到物理 Bank 的映射关系**。

在原始实现中，逻辑坐标$(x_l, y_l)$和物理坐标$(x_p, y_p)$都是相同的。例如，我想读取矩阵(smem)中的元素$(x_l, y_l)$，对应读取到物理内存中的元素$(x_p, y_p)$，在这里$(x_l, y_l)$与$(x_p, y_p)$相同。

而swizzling则对逻辑坐标进行了一次映射：$(x_p, y_p)=f(x_l,y_l)$。在写入过程中：`smem[ty][tx ^ ty] = input[id_y * nx + id_x];`，逻辑坐标是(tx, ty), 映射成物理坐标则变成了(tx^ty, ty)

既然我们写入smem时把逻辑坐标经过了一个映射才变成物理坐标，那么读取物理坐标的时候必须要把逻辑坐标进行逆映射才能保证正确性。于是，在读取阶段：`output[trans_y * ny + trans_x] = smem[tx][ty ^ tx];`

比如，我写入smem的一行，把列坐标进行了XOR映射。原本写入的应该是`smem[ty][tx]`，映射之后变成了`smem[ty][tx ^ ty]`。然后，我要读取smem的一列，原本读取的应该是`smem[tx][ty]`， 但是列坐标被映射过！所以实际读取的应该是`smem[tx][ty ^ tx]`，即对行左边进行了逆映射。



修改之后，我们再来看ncu：

![249dd511863ba3223943a3370a73a337](\images\swizzling避免bank_conflict\3.png)

效果拔群。

























