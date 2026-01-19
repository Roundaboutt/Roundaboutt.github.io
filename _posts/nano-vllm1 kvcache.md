# 从最基础的大模型推理过程开始讲起

## 传统KV计算的缺陷

我们来回顾一下，LLM推理最最原始的流程是什么样的（仅self-attention的计算部分）

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\1.jpg" alt="1" style="zoom: 67%;" />

首先，一个prompt先转换成一组token，也就是一个sequence。每个seq在做完词嵌入(Embedding)和旋转位置编码(RoPE)后，再分别与不同的权重矩阵相乘，得到了矩阵Q、K、V，维度为(batch_size, N, d)。其中batch_size是批量大小；N和d分别是上下文长度和词嵌入维度。

然后，QKV三个矩阵在d维度上分为m个头，于是维度变成了(batch_size, num_heads, N, d)，num_heads即为头数m。接下来就是进行一系列的注意力分数计算，每个头将各自计算得到的注意力分数矩阵O拼在一起，就得到了最终的结果。再经过后续的计算得到最终输出的token，并将其拼接到原始的seq后面形成一个新的seq，再将其全部输入self-attention做生成下一个token的计算。

那么问题来了：我们在计算完第 n+1 个token后，是不是需要把整个seq(长度为n+1)作为输入继续计算，但是这个seq的前 n 个token在上一轮计算中已经计算过了！于是，整个seq的计算只有第 n+1 个token是新的，而前 n 个token的计算全部都在重复！于是我们想，为什么不把前 n 个token计算得到的K、V矩阵存起来呢，这样不就省去了一大笔计算开销吗？于是KVCache就出现了！

---

## KVCache的原理

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\2.jpg" alt="2" style="zoom:67%;" />

由上图我们可以很清晰地看到，每次只需要输入上一步生成的token进行计算。当前输入的token计算注意力分数时直接调用之前的KVCache进行计算，省去了计算前 n 个token的KV矩阵的过程，并把当前token新生成的K、V矩阵追加到KVCache的末尾。

---

## KVCache的缺陷

接下来又有问题了：KVCache该设置成多大呢？当模型开始处理一个请求时，它不知道你会聊多久（生成 10 个字还是 1000 个字）。为了保险起见，它必须预先申请一块**巨大且连续**的内存。需要注意的是，每个请求（seq）都需要申请一块**巨大且连续**的内存。

<img src="F:\my_website\Roundaboutt.github.io\images\nano-vllm\3.jpg" alt="3" style="zoom:67%;" />

假设 max_seq_len=10，则我们需要为每个 seq 分配一块长度为 10 的连续内存。从图中可以看出，每个 seq 对各自内存的利用并不充分，当 seq 个数增加时，造成的内存浪费会越来越多。

因为不知道生成的序列到底会有多长，传统做法通常会按照模型的**最大上下文长度（Max Context Length）**或者一个很大的预设值来预分配显存。假设模型最大支持 2048 长度。 哪怕你只问了一句“你好”，占用了 2 个 token，剩下的 2046 个 token 的位置也被这一单请求给锁死了，其他请求根本用不了。在 vLLM 的论文里提到，传统系统的显存浪费极其严重，**甚至高达 60% - 80% 的显存都是被这种“预留但未被使用”的空白填满的**。 这叫**内部碎片（Internal Fragmentation）**。

---

## Paged Attention

针对这些缺陷，于是就有了Paged Attention。Paged Attention的主要思想来源于操作系统中的**虚拟内存**和**分页**技术。传统的KVCache的主要缺点在于：内存必须是连续的！而Paged Attention的解决方案就是：放弃内存的连续性。它把内存里的 KV Cache 切分成一个个固定大小的小块，叫**Block（块）**。比如，规定 1 个 Block 只能存 16 个 token 的 K 和 V 数据。

![图片来自论文](F:\my_website\Roundaboutt.github.io\images\nano-vllm\4.jpg)

我们可以看到，在逻辑上，每个 prompt 的 token 都是连续的，它们通过 block table 映射到不连续的内存块中。这样就做到了屋里内存中的不连续存储，也就彻底消除了外部碎片。逻辑内存相当于操作系统中的虚拟内存，每个block就是虚拟内存中的一个page。每个 block 的大小固定，在 vLLM 中默认大小为 16，即可存储 16 个 token 的 K/V 值。唯一可能产生内部碎片的情况在于，假如你有17个 token，在存入一个 block 后还剩一个 token，于是就智能



前面说了，在传统的KVCache中，每个 seq 都需要分配一块最大的连续内存。但是在Paged Attention中，所有的 seq 共用同一个KVCache。如下图所示，这是两个 seq 同时使用一个KVCache时的情景。

![image-20260118140916072](F:\my_website\Roundaboutt.github.io\images\nano-vllm\5.jpg)

我们来看看在nano-vllm中对KVCache的分配：

```python
def allocate_kv_cache(self):
    config = self.config
    hf_config = config.hf_config

    # 获取当前GPU的显存信息
    free, total = torch.cuda.mem_get_info()
    used = total - free

    # 从上次重置以来，current 达到过的峰值
    peak = torch.cuda.memory_stats()["allocated_bytes.all.peak"]
    # 当前 PyTorch 分配器实际分配给 Tensor 的总字节数
    current = torch.cuda.memory_stats()["allocated_bytes.all.current"]

    # 计算单个物理块的尺寸：
    num_kv_heads = hf_config.num_key_value_heads // self.world_size # 每个GPU上的KVheads
    #每个头的维度(每个头的向量有多长)                       hidden_size是embedding的维度, 把hidden_size平均分给每个头
    head_dim = getattr(hf_config, "head_dim", hf_config.hidden_size // hf_config.num_attention_heads)
    # 一个块的字节数 = (K和V) * 层数 * 每块token数 * 每层头数 * 每头维度 * 每个数字的字节数
    # 一个完整的物理块，它能够容纳 block_size 个 token 在模型所有 num_hidden_layers 层中的全部 Key 和 Value 数据，所需要的总字节数。
    block_bytes = 2 * hf_config.num_hidden_layers * self.block_size * num_kv_heads * head_dim * hf_config.torch_dtype.itemsize

    # 计算总共可以分配多少个块
    config.num_kvcache_blocks = int(total * config.gpu_memory_utilization - used - peak + current) // block_bytes
    assert config.num_kvcache_blocks > 0

    # 创建巨大的 KV 缓存 Tensor  第一个维度2用于区分是k还是v
    self.kv_cache = torch.empty(2, hf_config.num_hidden_layers, config.num_kvcache_blocks, self.block_size, num_kv_heads, head_dim)
    layer_id = 0
    # 遍历所有的子模块, 将KVCache的物理块绑定到模型的属性中
    for module in self.model.modules():
        # 找有k_cache 和 v_cache 接口的模块(注意力模块)
        if hasattr(module, "k_cache") and hasattr(module, "v_cache"):
            module.k_cache = self.kv_cache[0, layer_id]
            module.v_cache = self.kv_cache[1, layer_id]
            layer_id += 1
```

我们可以看到，KVCache的维度是：

```python
(2, hf_config.num_hidden_layers, config.num_kvcache_blocks, self.block_size, num_kv_heads, head_dim)
(K/V, 哪一层, 层中的哪一块, 块中的哪个token, token的哪一个head, 具体的数值)
```



**Paged Attention另一个高效的机制就是"Copy-on-Write"**

### parallel sampling

传统的KVCache在面临好几条相同的 seq 时会把每个 seq 的KV值都计算一遍，然后全部存起来。而实际上，根本就没必要存好几份相同的KV，这也是内存的浪费。

我们来看看 Paged Attention 会怎么做。

![image-20260118145643368](F:\my_website\Roundaboutt.github.io\images\nano-vllm\6.jpg)

我们输入了两个相同的 prompt："Four score and seven years ago our"，其中"Four score and seven"存储在 block7，"years ago our"存储在 block1，两个 seq 只需要存储一份KV值。此时 block1 和block7 的 Ref count（引用计数）都是2，代表一共有两个 seq 共用这个 block 中存储的KV。

接下来，prompt1 生成了下一个 token: "fathers"，而 prompt2 生成的下一个 token 是"mothers"。这时，整个 block1 中的内容都会复制到另一个 block中（这里假设是 block3）。现在，"mothers" 放入 block1 中，"fathers"放入 block3 中。block1 和 block3 的 Ref count 都变为1。

有了**Copy-on-Write**机制后，不管你要生成多少个回答，Prompt 的 KV Cache 永远只需要存一份。

如果你的 Prompt 有 2000 个 token，生成 10 个回答：

-   无 CoW： 显存占用 $2000 \times 10 = 20000$ tokens。

-   有 CoW： 显存占用 $2000 \times 1 = 2000$ tokens。

    节省了 90% 的显存！这意味着你可以塞进更多的并发请求（Batch Size 变大）。

    

### beam search

同样，Paged Attention 在 beam search 中也非常适用。

在传统的 Beam Search 中，我们每一轮都会保留得分最高的 k 个候选（即 Beam Width）。麻烦点在于：随着步数增加，不同的候选序列可能会有**共同的祖先**。为了保持每个候选序列的独立性，系统不得不为每一个 Beam 完整地复制一份 KV Cache。哪怕这 k 个序列的前 99% 都是一模一样的，也要在显存里存 k 遍。

而 Paged Attention 利用分页和 Copy-on-Write 机制完美地解决了这个问题。假设 beam width = 4：

当所有的候选序列都源自同一个 Prompt 时，它们在物理上**共用同一组物理块**。内存里只有一份 Prompt 的 KV Cache，所有的 Beam（Beam 1 到 Beam 4）的页表都指向相同的物理地址 `[Block 1, Block 2, ...]`，这些物理块的 `ref_count = 4`。

![image-20260119195634896](F:\my_website\Roundaboutt.github.io\images\nano-vllm\7.jpg)

并且在 decode 过程中，随着逻辑块 block 被淘汰，它占有的物理内存也被释放。Block 0、Block 1 和 Block 3，这三块内存在整个过程中只存了一份，却同时支撑着 4 个 Beam 的运算。如果没有 Paged Attention，则所有的block 都得复制 4 次。

---

## 代码详解

理论部分说完了，我们来看看 nano-vllm 中的代码是如何实现 Paged Attention 的。

在 block_manager.py 中，Class Block 表示一个物理块，Class BlockManager 用于对物理块的管理。

![image-20260119202423965](F:\my_website\Roundaboutt.github.io\images\nano-vllm\8.jpg)



我们先来看物理块 Block：

```python
class Block:
    # 一小块物理GPU内存, 可以想象成操作系统中的一个"内存页"
    def __init__(self, block_id):
        self.block_id = block_id    # 物理块唯一ID
        self.ref_count = 0          # 引用计数: 有多少sequence正在使用这个块
        self.hash = -1              # 内容哈希值: 用于快速识别和复用
        self.token_ids = []         # 存储的token ID, 用于哈希碰撞检测

    # 通过对块内的token内容进行哈希计算，管理器可以快速判断新来的请求前缀是否与缓存中已有的某个块内容完全相同。
    # 如果哈希匹配，就可以直接复用，避免了对这部分prompt的重复计算，极大地提升了处理相似请求时的速度。
    # token_ids 用于在极罕见的哈希碰撞情况下进行最终确认。
    
    def update(self, hash: int, token_ids: list[int]):
        self.hash = hash
        self.token_ids = token_ids

    def reset(self):
        self.ref_count = 1
        self.hash = -1
        self.token_ids = []

```





