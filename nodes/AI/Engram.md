---
name: Engram
field: AI
year: 2026
tags:
  - DeepSeek
  - n-gram
  - 哈希
  - embedding
  - 条件记忆
  - MoE
  - 稀疏
  - tokenizer
  - BPE
  - fastText
  - 神经科学
desc: DeepSeek 2026 提出的条件记忆模块：把 n-gram 哈希成下标，O(1) 查 embedding 表，替 Transformer 省下重算静态知识的层
layer: AI应用
---
# Engram

Engram 是 DeepSeek 2026 年 1 月在论文 *Conditional Memory via Scalable Lookup: A New Axis of Sparsity for Large Language Models* 里提出的模块：给 Transformer 挂一张巨大的 n-gram embedding 表，按当前位置结尾的 n-gram 哈希查表，O(1) 取出一段「静态记忆」加回模型。它把 n-gram 这个统计 NLP 时代的老东西，拿回来当大模型的查表记忆用。

## 为什么需要它
人名、实体、固定搭配这类静态知识，原来 Transformer 只能靠前几层 attention + FFN 一次次「算」出来——等于拿计算去模拟查表，白占深度。MoE 是「条件计算」（每个 token 只走部分专家），Engram 是「条件记忆」（每个 token 只查几行表），论文称之为稀疏化的第二条轴。

## 怎么运作
1. **tokenizer 压缩**：先把只差大小写、前导空格之类的 token 归并成同一个 ID，缩小有效词表
2. **取 n-gram**：对每个位置，取以它结尾的 2-gram、3-gram（token ID 组合）
3. **多头哈希查表**：每个 n-gram 用几组哈希函数映射成下标，去 embedding 表里取向量——允许冲突，不追求精确
4. **上下文门控**：用当前 hidden state 判断取回来的记忆和上下文搭不搭，不搭就压低，抵消哈希冲突和一词多义
5. 结果加回残差流；只插在部分层里，不是每层都有

工程上的关键：查哪一行只由输入 token 决定，**算之前就知道**，所以表可以放在 CPU 内存、提前 prefetch，不占 GPU 显存。

## 容易搞混的
- **和 n-gram 语言模型**：索引方式一脉相承（都是「连续 n 个 token」），但 n-gram LM 表里存计数、自己就是整个模型；Engram 表里存训练出来的向量，只是 Transformer 里的一个插件。
- **和 BPE / 分词**：BPE 在切词时就把高频组合并成一个 token，序列变短；Engram 不改序列，在旁边给组合查一个向量——效果上像不改 tokenizer 地扩了一套「多 token 词条」。
- **和 KV Cache 压缩**：没关系。千问说它是 DeepSeek-V3 的 KV Cache 压缩，是错的；V3 里也没有它。
- **名字**：engram 是神经科学里「记忆痕迹」的意思，同时谐音 n-gram。

## 我的理解
待补：……

## 线头
- n-gram 语言模型：索引方式的来源
- MoE：条件计算，和 Engram 的条件记忆是两条稀疏轴；论文讨论了稀疏参数怎么在两者之间分配（存在一个最优比例，具体数字待核）
- BPE / subword 分词：「多 token 单元在哪一层处理」的对比对象
- fastText（2016）：字符 n-gram 哈希成 embedding，思路上的前辈（Engram 论文是否直接引用，待核）
- 哈希技巧：用哈希代替显式大词表
- 神经科学：engram = 记忆痕迹
- DeepSeek-V3 / 后续模型：Engram 是否进了 DeepSeek 的正式模型，待核

## 关系
