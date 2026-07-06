# LLM（大语言模型）架构、原理与实现

## 一、什么是 LLM

> **LLM (Large Language Model)** 是一个超大规模的神经网络，通过海量文本训练，学会"预测下一个词"，从而具备理解、生成、推理等能力。

**一句话：** LLM = 巨大的 Transformer 解码器 + 海量数据训练 + 预测下一个 token。

---

## 二、核心架构：Transformer

### 2.1 整体结构

现在的主流 LLM（GPT 系列、LLaMA、DeepSeek 等）都使用 **Decoder-only Transformer** 架构。

```
输入文本 "今天天气"
       │
       ▼
  Tokenizer（分词器）
       │
       ▼
  Token Embedding（词向量）
       │
       ▼
  ┌─────────────────────────────┐
  │  Transformer Block × N 层    │  ← N 可以到几十甚至上百层
  │  ┌───────────────────────┐  │
  │  │  Multi-Head Attention │  │  ← 核心：理解上下文
  │  ├───────────────────────┤  │
  │  │  Feed-Forward Network │  │  ← 非线性变换
  │  ├───────────────────────┤  │
  │  │  Layer Norm + 残差连接 │  │  ← 稳定训练
  │  └───────────────────────┘  │
  └─────────────────────────────┘
       │
       ▼
  LM Head（线性层 → Softmax）
       │
       ▼
  输出: 下一个 token 的概率分布
  → 预测 "真" → "今天天气真"
  → 预测 "好" → "今天天气真好"
```

### 2.2 关键组件详解

#### ① Tokenizer（分词器）

把文本切成一个个 token（词元），是模型"认识"文字的方式。

```
中文: "今天天气真好"
  → ["今天", "天气", "真", "好"]  或  ["今", "天", "天", "气", "真", "好"]

英文: "Hello world"
  → ["Hello", " world"]

常用分词算法:
  BPE (Byte Pair Encoding)  → GPT-2/3/4, LLaMA
  SentencePiece             → T5, LLaMA
  WordPiece                 → BERT
```

#### ② Embedding（词嵌入）

把每个 token 映射成一个**高维向量**（比如 4096 维），语义相近的词，向量也相近。

```
"猫" → [0.12, -0.34, 0.87, ..., 0.05]   (4096 维)
"狗" → [0.15, -0.31, 0.82, ..., 0.08]   ← 和猫的向量很接近
```

#### ③ Attention（注意力机制）— 核心

LLM 能"理解上下文"全靠它。公式如下：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right) \cdot V$$

通俗理解：
- 每个词会"看"前面的所有词，决定哪些词和当前词**最相关**
- 相关性越高，权重越大，影响越大

```
输入: "我 昨天 在 公园 里 看到 一只 猫"
                            ↑
问: "猫" 最关注哪些词？
  → "看到"（权重 0.4）"公园"（权重 0.3）"一只"（权重 0.2）
  → 模型理解: 在公园里看到了一只猫
```

**Multi-Head Attention（多头注意力）：** 用多组 Q/K/V，从不同角度理解语义（语法、语义、位置等）。

**自回归 Mask：** Decoder-only 架构使用 Causal Mask（因果掩码），确保每个 token 只能看到**它前面的 token**，不能偷看后面的。

```
Token: 我 → 爱 → 吃 → 苹果
"我"    ✓    ✗    ✗    ✗
"爱"    ✓    ✓    ✗    ✗
"吃"    ✓    ✓    ✓    ✗
"苹果"  ✓    ✓    ✓    ✓
```

#### ④ Feed-Forward Network（前馈网络）

两层全连接 + 激活函数（通常用 SwiGLU），对每个 token 做非线性变换，增强表达能力。

```
FFN(x) = W2 · σ(W1 · x + b1) + b2

现代 LLM 常用 SwiGLU 变体:
FFN(x) = (W3 · σ(W1 · x)) ⊙ (W2 · x)
```

#### ⑤ Layer Normalization + 残差连接

```python
# 残差连接: 让梯度能直接流过，防止深层网络退化
output = x + Attention(LayerNorm(x))

# 现代 LLM 常用 RMSNorm（更高效）
output = x + Attention(RMSNorm(x))
```

---

## 三、训练原理

LLM 的训练分为三个阶段：

### 3.1 预训练 (Pre-training)

训练目标：**预测下一个 token**（Next Token Prediction）。

```
输入:  "今天 天气 真"
目标:  "好"

输入:  "1 + 1 ="
目标:  "2"

输入:  "def hello():"
目标:  "\n    print"
```

- **数据量：** 数万亿 token（几 TB 的文本）
- **数据来源：** 网页（Common Crawl）、书籍、代码、论文、维基百科等
- **损失函数：** 交叉熵损失（Cross-Entropy Loss）
- **训练时间：** 数千 GPU 跑几个月
- **成本：** GPT-4 级别模型训练成本约 $100M+

### 3.2 监督微调 (SFT — Supervised Fine-Tuning)

用高质量的**问答对**（instruction-tuning data）来训练，让模型学会"对话"。

```
[用户] 解释一下什么是光合作用
[助手] 光合作用是植物利用光能...（详细回答）

[用户] 写一首关于夏天的诗
[助手] 夏日炎炎蝉鸣响...（诗歌）
```

这个阶段让模型从"补全文本"变成"回答问题"。

### 3.3 对齐训练 (Alignment / RLHF)

让模型的输出**更有用、更诚实、更无害**（Helpful, Honest, Harmless）。

**RLHF（人类反馈强化学习）流程：**

```
步骤 1: 收集偏好数据
  模型生成多个回答 → 人类标注员选最好的那个

步骤 2: 训练奖励模型 (Reward Model)
  学会"打分"，判断哪个回答更好

步骤 3: PPO 强化学习
  用奖励模型优化 LLM，让它的输出得分更高
```

**DPO (Direct Preference Optimization)** — 更新的方法，不需要单独训练奖励模型，直接从偏好数据中优化，更简单高效。

---

## 四、关键技术细节

### 4.1 位置编码 (Positional Encoding)

Transformer 本身不理解顺序，需要告诉模型每个 token 的位置。

```
传统: 正弦位置编码（原始 Transformer）
现代: RoPE（旋转位置编码）— LLaMA、Qwen、DeepSeek 等主流模型都在用
```

RoPE 的核心思想：通过旋转向量来表示位置，具有很好的**外推性**（训练时 4K 上下文，推理时可以用到 32K+）。

### 4.2 KV Cache（推理加速）

生成文本时，不需要每步都重新计算所有 token 的注意力。

```
第 1 步: 计算 "今天" → 存 K,V
第 2 步: 计算 "天气" → 只算新 token，复用之前存的 K,V
第 3 步: 计算 "真"   → 同上
...
```

这使得推理速度从 O(n²) 降到 O(n)，能快几十倍。

### 4.3 模型量化 (Quantization)

把 32-bit 浮点参数压缩到 4-bit / 8-bit，大幅减少显存占用。

```
FP16:  7B 模型 ≈ 14GB 显存
INT4:  7B 模型 ≈ 4GB 显存    ← 普通电脑也能跑！

常用工具: llama.cpp (GGUF), GPTQ, AWQ, bitsandbytes
```

### 4.4 MoE（混合专家模型）

不是所有参数每次都用，而是每次只激活一部分"专家"。

```
传统架构: 全部参数参与每次计算
MoE 架构: 每次只激活 10%-20% 的参数

DeepSeek-V3: 671B 总参数，每次只激活 37B  → 推理成本大幅降低
Mixtral 8×7B: 47B 总参数，每次只激活 13B
```

---

## 五、实现流程（从零到推理）

### 5.1 训练流程

```
步骤 1: 数据准备
  收集原始文本 → 清洗（去重、过滤低质量）→ Tokenize → 打包成训练 batch

步骤 2: 预训练
  分布式训练（数据并行 + 模型并行 + 流水线并行）
  常用框架: Megatron-LM, DeepSpeed, FSDP

步骤 3: 监督微调
  准备高质量对话数据 → 全量/部分参数微调（LoRA/QLoRA 省钱）

步骤 4: 对齐
  RLHF 或 DPO → 模型更听话

步骤 5: 评估
  跑基准测试: MMLU, GSM8K, HumanEval 等
```

### 5.2 推理（生成文本）

模型生成文本的过程叫**自回归（Autoregressive）解码**：

```python
# 伪代码：LLM 推理过程
def generate(prompt, max_tokens=100):
    tokens = tokenizer.encode(prompt)      # 1. 分词
    
    for _ in range(max_tokens):
        logits = model(tokens)             # 2. 前向传播
        probs = softmax(logits[-1])        # 3. 取最后一个 token 的概率
        
        # 4. 采样策略
        next_token = sample(probs, temperature=0.7, top_p=0.9)
        
        tokens.append(next_token)           # 5. 追加到序列
        
        if next_token == EOS:              # 6. 遇到结束符就停
            break
    
    return tokenizer.decode(tokens)        # 7. 还原为文本
```

### 5.3 采样策略

| 策略 | 说明 | 效果 |
|------|------|------|
| **Greedy** | 每次选概率最高的 token | 确定性强，容易重复 |
| **Temperature** | 调低 → 更确定 / 调高 → 更随机 | 控制"创造力" |
| **Top-K** | 只从概率最高的 K 个中选 | 过滤低质量选项 |
| **Top-P (Nucleus)** | 累积概率达到 P 就截断 | 动态调整候选数 |

---

## 六、主流模型速览

| 模型 | 参数量 | 架构特点 | 开源 |
|------|--------|---------|------|
| GPT-4 | ~1.8T（未公开） | Dense Transformer | ❌ |
| GPT-4o | 未公开 | 多模态（文本+图像+语音） | ❌ |
| Claude 3.5 | 未公开 | Constitutional AI | ❌ |
| LLaMA 3 | 8B / 70B / 405B | Dense, RoPE, SwiGLU | ✅ |
| DeepSeek-V3 | 671B (37B 激活) | MoE, Multi-head Latent Attention | ✅ |
| Qwen 2.5 | 0.5B~72B | Dense, RoPE, SwiGLU | ✅ |
| Mistral | 7B / 8×7B | MoE, Sliding Window Attention | ✅ |
| Gemma 2 | 2B / 9B / 27B | Dense, GeGLU | ✅ |

---

## 七、扩展话题

### 7.1 RAG（检索增强生成）

让 LLM 能"查资料"再回答，解决知识截止日期和幻觉问题。

```
用户提问 → 检索相关文档 → 拼接到 prompt → LLM 生成带引用的回答
```

### 7.2 Agent（智能体）

让 LLM 能**调用工具**（搜索、代码执行、API 调用等），不再只是"说话"。

```
用户: "帮我查一下今天北京天气，然后发邮件给老板"
  → LLM 调用天气 API → 拿到结果 → 调用邮件 API → 发送
```

### 7.3 长上下文

现代 LLM 支持超长上下文窗口：

| 模型 | 上下文长度 |
|------|-----------|
| GPT-4 Turbo | 128K |
| Claude 3 | 200K |
| Gemini 1.5 Pro | 1M+ |
| Qwen 2.5 | 128K |

---

## 八、一句话总结

> LLM 的本质是一个**超级 autocomplete**：你给它一段文本，它预测下一个最可能的 token，不断重复这个过程，就能生成连贯的回复。它的"智能"来源于海量数据中学习到的**语言模式**和**世界知识**。

---

## 参考资源

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — Transformer 原始论文
- [LLaMA: Open and Efficient Foundation Language Models](https://arxiv.org/abs/2302.13971) — LLaMA 论文
- [DeepSeek-V3 Technical Report](https://arxiv.org/abs/2412.19437) — MoE 架构详解
- [Andrej Karpathy - Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) — 手把手实现 GPT 的视频教程