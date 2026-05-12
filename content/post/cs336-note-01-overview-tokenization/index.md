+++

title = "CS336 Note 01: Overview, Tokenization"
date = 2026-05-12T16:42:15+08:00
slug = "cs336-note-01-overview-tokenization"
description = "构建大语言模型的完整链路，涵盖模型架构、系统优化与数据对齐，并解析了 BPE 分词算法的底层代码实现"
tags = ["LLM"]
categories = ["Notes"]
image = "cs336_header.webp"

+++

[CS 336](http://cs336.stanford.edu/spring2025/) 课程笔记，仅供参考。

## 1. Overview

### 1.1 basic

构建一个完整的语言模型由三步组成：tokenization，模型结构和训练方法

#### 1.1.1 Tokenization

tokenizer 可以实现字符串和 int 序列（tokens）之间的转换

{{<figure src="36fbaf2d02a5e921b94330964cf875c7.webp" title="tokenizer" width=800 >}}

本课程主要讲解的是 Byte-Pair Encoding（BPE）tokenizer  [Sennrich+ 2015](https://arxiv.org/abs/1508.07909)

此外还有一个无需 tokenizer 的方法，但是目前并没有起到显著成效  [Xue+ 2021](https://arxiv.org/abs/2105.13626), [Yu+ 2023](https://arxiv.org/pdf/2305.07185.pdf), [Pagnoni+ 2024](https://arxiv.org/abs/2412.09871), [Deiseroth+ 2024](https://arxiv.org/abs/2406.19223)

#### 1.1.2 结构

{{<figure src="ff893a3e2c5ef8e817bc78ee964ecce4.webp" title="transformer architecture" width=600 >}}

在 transformer 基础上又有一些变种：

- 损失函数：ReLU，SwiGLU [Shazeer 2020](https://arxiv.org/pdf/2002.05202.pdf)
- 位置编码：sinusoidal，RoPE [Su+ 2021](https://arxiv.org/pdf/2104.09864.pdf)
- 归一化：LayerNorm，RMSNorm [Zhang+ 2019](https://arxiv.org/abs/1910.07467)
- 归一化层的位置：pre-norm vs post-norm [Xiong+ 2020](https://arxiv.org/pdf/2002.04745.pdf)
- MLP 层：混合专家模式 [Shazeer+ 2017](https://arxiv.org/pdf/1701.06538.pdf)
- 注意力：滑动窗，线性注意力 [Jiang+ 2023](https://arxiv.org/pdf/2310.06825.pdf), [Katharopoulos+ 2020](https://arxiv.org/abs/2006.16236)
- 低维注意力：GQA，MLA [Ainslie+ 2023](https://arxiv.org/pdf/2305.13245.pdf), [DeepSeek-AI+ 2024](https://arxiv.org/abs/2405.04434)
- - State-space models: Hyena  [Poli+ 2023](https://arxiv.org/abs/2302.10866)

#### 1.1.3 训练方法

- 优化器：AdamW，Muon，SOAP [Kingma+ 2014](https://arxiv.org/pdf/1412.6980.pdf), [Loshchilov+ 2017](https://arxiv.org/pdf/1711.05101.pdf), [Keller 2024](https://kellerjordan.github.io/posts/muon/), [Vyas+ 2024](https://arxiv.org/abs/2409.11321)
- 学习率缩放：cosine，WSD [Loshchilov+ 2016](https://arxiv.org/pdf/1608.03983.pdf), [Hu+ 2024](https://arxiv.org/pdf/2404.06395.pdf)
- Batch size：critical batch size [McCandlish+ 2018](https://arxiv.org/pdf/1812.06162.pdf)
- 正则化：dropout，weight decay
- 超参数：head number，hidden dimension

### 1.2 系统

#### Kernels

{{<figure src="61e278fe366b1e97faf4b0dc2dbe9cab.webp" title="dram and sram" width=600 >}}

在 GPU 内部，DRAM 和 SRAM 之间进行数据传输，需要最大化数据传输的能效

#### 1.2.2 Parallelism

{{<figure src="c294334d9b5eb4d4156ecf56977ff656.webp" title="gpu group" width=800 >}}

如果有一个 GPU 集群，数据在 GPU 之间的移动将会比 GPU 内部更为低效

- 进行 collective operation：gather，reduce，all-reduce
- GPU 数据分片（参数，激活状态，梯度，优化器状态）
- 分布计算：数据、tensor、pipeline、sequence 并行

#### 1.2.3 推理

一般来说，推理计算比训练计算需求更高

{{<figure src="3cd276051a37e9545ed47bb02ae50e6b.webp" title="inference phase" >}}

推理可以分为 prefill 和 decode 两个阶段

- prefill：给定 token，可以同时处理所有token
- decode：需要递归依次生成token

提高 decode 效率的方法：

- 使用更加轻量的模型（模型剪枝、量化、蒸馏）
- speculative decoding：使用轻量模型初步生成一些 token，然后再通过全量模型打分
- 系统优化：KV cache，batch 化

### 1.3 Scaling Laws

进行小规模的实验，来预测大规模场景下的超参数和损失

给定一定的 FLOPs 资源，训练一个更大的模型更好，还是使用更多的 token 训练更好

Compute-optimal scaling laws:  [Kaplan+ 2020](https://arxiv.org/pdf/2001.08361.pdf), [Hoffmann+ 2022](https://arxiv.org/pdf/2203.15556.pdf)

{{<figure src="ab0a2a8d355ce9da8c2c263a514c8c19.webp" title="scaling law" width=800 >}}

### 1.4 Data

我们希望模型具有什么样的能力：多语言、编程、数学

#### 1.4.1 模型评估

- 困惑度 perplexity：大模型评估的基础方式
- 标准测试：M M L U，HellaSwag，GSM8K
- 指令跟随：ALpacaEval，IFEval，WildBench
- LM-as-a-judge
- 全系统评估：RAG，agent

#### 1.4.2 数据筛选

- 数据来源：互联网上爬虫得到的网页，数据，arXiv 论文，Github 代码等
- 可能需要数据授权
- 格式：HTML，PDF，文件夹

#### 1.4.3 数据处理

- 格式转换：将 HTML/PDF 转换成文本
- 过滤：过滤掉有害内容
- 去重：降低计算量，防止模型 memorization

### 1.5 对齐

经过预训练后，模型已经非常擅长预测下一个 token，但是需要通过对齐让模型真正可用

- 让语言模型可以遵守对指令做出响应的范式
- 优化响应风格
- 防止产生有害内容

对齐可以分为两个阶段：监督学习微调和反馈强化学习

#### 1.5.1 监督学习微调 SFT

使用*指令数据 instruction data* `<prompt, response>` 对进行监督学习训练

```python
sft_data: list[ChatExample] = [
	ChatExample(
		turns=[
			Turn(role="system", content="You are a helpful assistant."),
			Turn(role="user", content="What is 1 + 1?"),
			Turn(role="assistant", content="The answer is 2."),
		],
	),
]
```

#### 1.5.2 Preference Data

经过指令数据的有监督训练，我们已经有了一个初步的可以跟随指令的模型，我们可以通过使用 preference data 训练让模型更加强大，而不需要更多的指令标注数据

preference data：让多个模型对同一个 prompt 给出响应，用户给出偏好

```python
preference_data: list[PreferenceExample] = [
    PreferenceExample(
        history=[
            Turn(role="system", content="You are a helpful assistant."),
            Turn(role="user", content="What is the best way to train a language model?"),
        ],
        response_a="You should use a large dataset and train for a long time.",
        response_b="You should use a small dataset and train for a short time.",
        chosen="a",
    )
]
```

- Proximal Policy Optimization (PPO) from reinforcement learning [Schulman+ 2017](https://arxiv.org/pdf/1707.06347.pdf) [Ouyang+ 2022](https://arxiv.org/pdf/2203.02155.pdf)
- Direct Policy Optimization (DPO): for preference data, simpler  [Rafailov+ 2023](https://arxiv.org/pdf/2305.18290.pdf)
- Group Relative Preference Optimization (GRPO): remove value function [Shao+ 2024](https://arxiv.org/pdf/2402.03300.pdf)

## 2. Tokenization

一个 tokenizer 需要实现编码和解码的功能：

- 编码：将字符串编码成 token（int 类型的索引）
- 解码：将 token 转换回字符串

其中所有可能的 token 的个数称为 vocabulary size

### 2.1 Character-based tokenization

一个字符串由一序列字符组成，每个字符可以直接被转换成整数索引值作为 token。例如，直接使用 `ord` 可以得到字符的 ASCII 码，可以作为一个简单的tokenizer

缺点：

- Unicode 字符的数量大概是 150K，导致 vocabulary size 非常大
- 很多字符在句子中很少使用，造成了稀疏性，vocabulary 非常低效

### 2.2 Byte-based tokenization

Unicode 字符串也可以直接用一系列的 byte 表示，这些 byte 可以使用 0 到 255 的整数索引

```python
assert bytes("a", encoding="utf-8") == b"a"
assert bytes("🌍", encoding="utf-8") == b"\xf0\x9f\x8c\x8d"
```

只用 255 种 byte 就可以表示所有的字符窗，vocabulary size 是 255。但是一个 byte 对应了一个整数 token，压缩率为 1, tokenization 之后得到的 token 序列非常长，造成自注意力计算量非常大

### 2.3 Word-based tokenization

NLP 模型中常用的是基于单词的 tokenization，首先将一个句子划分成单词序列，然后将每个可以划分的单词指定一个整数索引，就可以建立一个tokenizer

```python
string = "I'll say supercalifragilisticexpialidocious!"
segments = regex.findall(r"\w+|.", string)
# 输出 ["i", "ll", "say", "supercalifragilisticexpialidocious"]
```

缺点：

- 单词的总数非常大
- 很多单词使用率非常低，模型很难充分学习
- 很难确定一个固定大小的词典，存在没有见过的词

### 2.4 Byte Pair Encoding

首先将字符串表示成 byte，然后让 tokenizer 自动选择 byte 组成单词（不一定是完整的人类单词），那么一些常见的单词将可以使用一个 token 表示，对于少见或者没有见过的单词可以使用多个 token 组合表示

```python
def train_bpe(string: str, num_merges: int) -> BPETokenizerParams:
	"""
    Start with the list of bytes of string.
    indices = list(map(int, string.encode("utf-8")))
    merges: dict[tuple[int, int], int] = {}
    vocab: dict[int, bytes] = {x: bytes([x]) for x in range(256)}
    """
    for i in range(num_merges):
    	"""
        Count the number of occurrences of each pair of tokens
        """
        # 首先计算每两个byte的两两组合，以及出现的频率
        counts = defaultdict(int)
        for index1, index2 in zip(indices, indices[1:]):
            counts[(index1, index2)] += 1

        # Find the most common pair，确定可以合并的byte
        pair = max(counts, key=counts.get)
        index1, index2 = pair

        # Merge that pair
        new_index = 256 + i
        merges[pair] = new_index
        vocab[new_index] = vocab[index1] + vocab[index2]
        indices = merge(indices, pair, new_index)

    return BPETokenizerParams(vocab=vocab, merges=merges)
```
