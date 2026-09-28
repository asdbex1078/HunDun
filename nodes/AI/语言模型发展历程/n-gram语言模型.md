---
name: n-gram语言模型
field: AI
year: 1948
tags:
  - NLP
  - 统计语言模型
  - 马尔可夫链
  - Markov
  - 信息论
  - Shannon
  - 语音识别
  - Jelinek
  - 平滑
  - Kneser-Ney
  - 统计机器翻译
  - 神经语言模型
  - 分词
  - Engram
desc: 用前 n−1 个 token 预测下一个的计数型语言模型；统计 NLP 时代的标准做法，死穴是稀疏和不懂近义
layer: AI应用
---
n-gram 语言模型用「前 n−1 个 token」近似整段历史来预测下一个 token：P(w_t ｜ 全部历史) ≈ P(w_t ｜ 前 n−1 个)，概率直接从语料里数出来。它是统计 NLP 时代语言模型的标准做法，也是神经语言模型要取代的那一代。

## 为什么需要它
「一整句话出现的概率」没法直接算——整句几乎都是语料里没见过的。n-gram 用马尔可夫假设把历史截断到 n−1 个词，问题就变成查一张计数表。1970s–2000s 的语音识别（IBM Jelinek 组）、拼写纠错、统计机器翻译里的语言模型组件都是它，常用 3-gram 到 5-gram。

## 怎么运作
以 n=2（bigram）为例。语料：`<s> I love NLP and I love AI </s>`（`<s>` `</s>` 是句首句尾标记）

1. 切成 token：`I / love / NLP / and / I / love / AI`
2. 相邻两两配对并计数：`(I,love)×2`、`(love,NLP)×1`、`(love,AI)×1`、`(NLP,and)×1`、`(and,I)×1`，外加 `(<s>,I)`、`(AI,</s>)`
3. 最大似然估计：P(后词 ｜ 前词) = count(前词,后词) / count(前词)
   - P(love ｜ I) = 2/2 = 1.0
   - P(NLP ｜ love) = P(AI ｜ love) = 1/2
4. 预测：看到 `love`，下一个词 50% 是 NLP、50% 是 AI；`love coffee` 没出现过，概率是 0。

n 取不同值：

| n | 名称 | 看多少上下文 | 在上面语料里 | 代价 |
|---|---|---|---|---|
| 1 | unigram | 不看，只看词频 | P(I)=P(love)=2/7，其余各 1/7；生成时和前文无关 | 没有上下文，只是一张词频表 |
| 2 | bigram | 前 1 个 | P(love ｜ I)=1.0 | 只看得到紧挨着的一个词 |
| 3 | trigram | 前 2 个 | P(and ｜ love NLP)=1.0，但 `love NLP` 只出现过 1 次 | 组合数随 n 指数增长，绝大部分格子是 0 |

n 越大上下文越足、表越稀疏。实践中语音识别常用 trigram，统计机器翻译常用 4–5-gram（配合平滑）。

## 容易搞混的
- **n-gram 不是分词**：分词决定「一个 token 是什么」，n-gram 是在切好的 token 序列上数「连续 n 个 token」。这 n 个单位可以是词、subword，也可以是字母（字符级 n-gram）。
- **稀疏靠平滑和回退补**：add-one、Good-Turing、Katz backoff（1987）、Kneser-Ney（1995）——没见过的组合不给 0，退回更短的 (n−1)-gram 去估。
- **和神经语言模型**：n-gram 里 `love` 和 `like` 是两个毫无关系的符号；Bengio 2003 的 NNLM 用词向量让相近的词共享统计，这是它被取代的根本原因。

## 我的理解
待补：……

## 线头
- 数学：Markov 1913 用马尔可夫链统计《叶甫盖尼·奥涅金》的字母序列，马尔可夫链第一次用在语言上
- 信息论：Shannon 1948《A Mathematical Theory of Communication》用 n 阶近似生成英文；1951《Prediction and Entropy of Printed English》估计英文的熵
- 语音识别：IBM Jelinek 组 1970s–80s 把 trigram 用进语音识别
- 平滑：Katz backoff（1987）、Kneser-Ney（1995）
- 统计机器翻译：语言模型组件，常用 4–5-gram
- 神经语言模型：Bengio 2003 NNLM，用词向量解决「不懂近义」
- 分词：BPE / subword——和 n-gram 是两件事，但都在处理「多个单位组成的组合」
- Engram：DeepSeek 2026 把 n-gram 拿回来，做成大模型里的哈希查表记忆

## 关系
- 演化为:: [[Engram]]
