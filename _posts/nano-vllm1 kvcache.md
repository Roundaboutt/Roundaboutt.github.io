# 从最基础的大模型推理过程开始讲起

## 传统KV计算的缺陷

我们来回顾一下，LLM推理最最原始的流程是什么样的（仅self-attention的计算部分）

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\1.jpg" alt="1" style="zoom: 67%;" />

首先，一个prompt先转换成一组token，也就是一个sequence。每个seq在做完词嵌入(Embedding)和旋转位置编码(RoPE)后，再分别与不同的权重矩阵相乘，得到了矩阵Q、K、V，维度为(batch_size, N, d)。其中batch_size是批量大小；N和d分别是上下文长度和词嵌入维度。

然后，QKV三个矩阵在d维度上分为m个头，于是维度变成了(batch_size, num_heads, N, d)，num_heads即为头数m。接下来就是进行一系列的注意力分数计算，每个头将各自计算得到的注意力分数矩阵O拼在一起，就得到了最终的结果。再经过后续的计算得到最终输出的token，并将其拼接到原始的seq后面形成一个新的seq，再将其全部输入self-attention做生成下一个token的计算。

那么问题来了：我们在计算完第 n+1 个token后，是不是需要把整个seq(长度为n+1)作为输入继续计算，但是这个seq的前 n 个token在上一轮计算中已经计算过了！于是，整个seq的计算只有第 n+1 个token是新的，而前 n 个token的计算全部都在重复！于是我们想，为什么不把前 n 个token计算得到的K、V矩阵存起来呢，这样不就省去了一大笔计算开销吗？于是KVCache就出现了！

## KVCache的原理

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\2.jpg" alt="2" style="zoom:67%;" />

由上图我们可以很清晰地看到，每次只需要输入上一步生成的token进行计算。当前输入的token计算注意力分数时直接调用之前的KVCache进行计算，省去了计算前 n 个token的KV矩阵的过程，并把当前token新生成的K、V矩阵追加到KVCache的末尾。



## KVCache的缺陷

接下来又有问题了：KVCache该设置成多大呢？当模型开始处理一个请求时，它不知道你会聊多久（生成 10 个字还是 1000 个字）。为了保险起见，它必须预先申请一块**巨大且连续**的内存。需要注意的是，每个请求（seq）都需要申请一块**巨大且连续**的内存。

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\3.jpg" alt="3" style="zoom:67%;" />

假设 max_seq_len=10，则我们需要为每个 seq 分配一块长度为 10 的连续内存。从图中可以看出，每个 seq 对各自内存的利用并不充分，当 seq 个数增加时，造成的内存浪费会越来越多。

因为不知道生成的序列到底会有多长，传统做法通常会按照模型的**最大上下文长度（Max Context Length）**或者一个很大的预设值来预分配显存。假设模型最大支持 2048 长度。 哪怕你只问了一句“你好”，占用了 2 个 token，剩下的 2046 个 token 的位置也被这一单请求给锁死了，其他请求根本用不了。在 vLLM 的论文里提到，传统系统的显存浪费极其严重，**甚至高达 60% - 80% 的显存都是被这种“预留但未被使用”的空白填满的**。 这叫**内部碎片（Internal Fragmentation）**。



## Paged Attention

针对这些缺陷，于是就有了Paged Attention。Paged Attention的主要思想来源于操作系统中的**虚拟内存**和**分页**技术。传统的KVCache的主要缺点在于：内存必须是连续的！而Paged Attention的解决方案就是：放弃内存的连续性。它把内存里的 KV Cache 切分成一个个固定大小的小块，叫**Block（块）**。比如，规定 1 个 Block 只能存 16 个 token 的 K 和 V 数据。

![图片来自论文](F:\my_website\Roundaboutt.github.io\images\nano-vllm\4.jpg)

我们可以看到，在逻辑上，每个 prompt 的 token 都是连续的，它们通过 block table 映射到不连续的内存块中。这样就做到了屋里内存中的不连续存储，也就彻底消除了外部碎片。逻辑内存相当于操作系统中的虚拟内存，每个block就是虚拟内存中的一个page。每个block的大小固定，在vLLM中默认大小为16，即可存储16个token的K/V值。









