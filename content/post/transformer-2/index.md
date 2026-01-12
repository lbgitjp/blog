---
title: Transformer/GPT架构背后的数学原理
description: 这篇文章旨在为熟悉代码的开发者提供一份深入的数学指南，帮助大家从底层的数学视角理解 Transformer 和 GPT 架构。
slug: transformer-2
date: 2026-01-10 08:00:00+0000
categories:
    - transformer
    -
tags:
    - 大语言模型
    - LLM
    - 数学
# 参考：
#
---

# 从代码到公式：Transformer/GPT 架构背后的数学原理

如果你已经阅读过 PyTorch 或 TensorFlow 中的 Transformer 源码（例如 `minGPT` 或 `nanoGPT`），你会发现代码中充满了 `nn.Linear`、`matmul`、`softmax` 等操作。如果你能熟练使用这些 API，说明你已经掌握了 LLM 的“骨架”。

然而，要真正理解模型的“灵魂”——为什么它能理解语言、为什么要除以根号 d、为什么要用旋转位置编码——我们需要深入到其背后的数学基石。

Transformer 的数学本质，是**线性代数**（处理数据表征）、**概率论**（处理不确定性和预测）与**微积分**（处理训练优化）的优雅结合。以下我们将按照数据在模型中的流向，逐一拆解其中的数学知识。

---

### 1. 基础基石：线性代数 (Linear Algebra)

LLM 约 90% 的计算量都消耗在矩阵乘法（Matrix Multiplication）上，这是模型处理信息的根本方式。

#### 1.1 向量与矩阵
*   **Token 的几何意义**：在代码中，一个单词（token）对应一个长度为 $d_{model}$（如 4096）的一维数组。在数学上，这是一个高维空间中的**向量** $\mathbf{x} \in \mathbb{R}^{d_{model}}$。每一个维度都代表了某种语义特征。
*   **Batch 处理**：为了利用 GPU 的并行计算能力，我们将 $B$ 个句子、每个句子 $T$ 个 token 堆叠，形成一个张量 $(B, T, d_{model})$。在数学推导中，我们通常关注单层的矩阵操作，记作输入矩阵 $X \in \mathbb{R}^{T \times d}$。

#### 1.2 线性变换 (Linear Projection)
代码中无处不在的 `nn.Linear(in_features, out_features)`，对应数学上的仿射变换：
$$ \mathbf{y} = \mathbf{x}W + \mathbf{b} $$
*   $\mathbf{x}$：输入行向量。
*   $W$：权重矩阵（形状 $d_{in} \times d_{out}$）。
*   $\mathbf{b}$：偏置向量。
*   **数学意义**：这不仅仅是乘法，而是将数据从一个特征空间**映射**（Project）到另一个特征空间。例如，从 4096 维的“嵌入空间”映射到 12288 维的“前馈网络空间”以提取更复杂的特征。

---

### 2. 几何与三角学：位置编码 (Positional Encoding)

Self-Attention 机制在数学上是“置换不变”的（Permutation Invariant），即打乱集合中元素的顺序，计算结果不变。为了让模型理解“语序”，必须引入几何或三角学的方法来注入位置信息。

#### 2.1 绝对位置编码 (Sinusoidal - 原始 Transformer)
利用三角函数的周期性，为每个位置生成唯一的指纹。公式如下：
$$ PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right) $$
$$ PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right) $$
*   **数学直觉**：这相当于用不同频率的波形来编码位置。更重要的是，对于固定偏移 $k$，位置 $pos+k$ 的编码向量可以表示为位置 $pos$ 向量的线性变换（旋转），这让模型容易学习相对位置关系。

#### 2.2 旋转位置编码 (RoPE - Llama/Qwen 等现代 LLM)
在现代大模型中，我们将向量视为**复平面**上的点。
*   **数学核心**：利用欧拉公式 $e^{i\theta} = \cos\theta + i\sin\theta$，将向量两两分组，通过旋转矩阵 $R_{\theta}$ 进行变换：
    $$ f(x, m) = x e^{im\theta} $$
*   **作用**：通过数学上的“旋转”操作，当两个 token 进行点积（Attention 计算）时，它们的数值结果只与它们的**相对距离** $(m-n)$ 有关，而与绝对位置无关。这是 RoPE 优于绝对位置编码的关键数学特性。

---

### 3. 核心机制：自注意力 (Self-Attention)

这是 Transformer 中数学密度最高的部分。

#### 3.1 投影 (Q, K, V)
输入 $X$ 经过线性变换分解为三个分量：
$$ Q = X W_Q, \quad K = X W_K, \quad V = X W_V $$

#### 3.2 缩放点积注意力 (Scaled Dot-Product Attention)
经典公式：
$$ \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V $$

我们来拆解其中的数学原理：

1.  **点积 (Dot Product) $QK^T$**：
    *   **几何意义**：向量点积 $A \cdot B = |A||B|\cos\theta$ 衡量了两个向量的**方向相似度**。
    *   **直觉**：模型在计算“Query（查询）”与所有“Key（键）”的匹配程度。点积越大，两者越相关。

2.  **缩放 (Scaling) $\frac{1}{\sqrt{d_k}}$**：
    *   **统计学原理**：假设 $Q$ 和 $K$ 的元素是独立同分布（i.i.d.）的随机变量，均值为 0，方差为 1。那么它们的点积 $Q \cdot K$ 的均值虽为 0，但方差会膨胀到 $d_k$（维度）。
    *   **数值稳定性**：当维度 $d_k$ 很大时（如 128），点积结果会非常大。这会导致 Softmax 函数进入“饱和区”，梯度趋近于 0，导致模型无法训练。除以 $\sqrt{d_k}$ 正是为了将方差拉回 1。

3.  **Softmax 函数**：
    $$ \text{softmax}(z_i) = \frac{e^{z_i}}{\sum_{j} e^{z_j}} $$
    *   **作用**：将一组任意实数（Logits）映射为**概率分布**。这决定了在构建当前语义时，应该给上下文中每个单词分配多少“关注度”（权重和为 1）。

4.  **加权求和 ($\dots \times V$)**：
    *   最后一步是矩阵乘法，本质上是 Value 向量的**期望**（加权平均）。

---

### 4. 函数理论：非线性激活 (Non-linear Activation)

如果网络中只有矩阵乘法，无论叠加多少层，模型最终都等价于一个单层线性模型（$W_2(W_1x) = W_{new}x$）。为了拟合复杂函数，必须引入非线性。

#### GELU / Swish (SiLU)
现代 LLM（如 Llama, GPT-3）很少使用 ReLU，而是用 GELU 或 Swish。
*   **Swish (SiLU) 公式**：
    $$ f(x) = x \cdot \sigma(x) = \frac{x}{1 + e^{-x}} $$
*   **数学特性**：
    *   **平滑性**：处处可导，比 ReLU 的折线更平滑。
    *   **非单调性**：允许负输入有少量的负输出，这种特性被证明有助于深层网络的梯度传播。

---

### 5. 统计学：归一化 (Normalization)

对应代码中的 `LayerNorm` 或 `RMSNorm`。

#### Layer Normalization
$$ \hat{x} = \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} \cdot \gamma + \beta $$
*   **统计操作**：计算当前样本特征向量的均值 $\mu$ 和方差 $\sigma^2$，进行标准化（Z-score）。
*   **几何意义**：将所有样本向量拉回到同一个分布范围内，防止在深层网络计算中数值发散（爆炸或消失）。

#### RMSNorm (Llama 常用)
去掉了均值项，只保留均方根（RMS）缩放：
$$ \bar{a}_i = \frac{a_i}{\text{RMS}(a)} g_i, \quad \text{其中 } \text{RMS}(a) = \sqrt{\frac{1}{n} \sum a_i^2} $$
*   **优势**：计算量更小，且实验表明在 LLM 训练中，控制数据的“幅度”（RMS）比控制“平移”（均值）更为重要。

---

### 6. 概率论：输出与损失

#### 下一个 Token 的预测 (Next Token Prediction)
LLM 的本质是在建模条件概率分布：
$$ P(w_1, w_2, \dots, w_T) = \prod_{t=1}^{T} P(w_t | w_1, \dots, w_{t-1}) $$

#### 交叉熵损失 (Cross-Entropy Loss)
训练的目标是最小化预测分布与真实分布的距离（KL 散度）。
$$ L = -\sum_{c} y_{true, c} \log(p_{pred, c}) $$
由于真实标签 $y_{true}$ 是 One-hot 向量（只有一个位置是 1），公式简化为：
$$ L = -\log(p_{correct\_token}) $$
*   **直观理解**：如果你对正确答案的预测概率 $p$ 越接近 1，损失 $L$ 就越接近 0；如果 $p$ 很低，损失就会趋向无穷大。

---

### 7. 微积分：优化 (Optimization)

虽然推理代码中不体现，但模型的训练完全依赖于微积分。

#### 梯度下降与链式法则
参数更新遵循梯度下降公式：
$$ \theta_{new} = \theta_{old} - \alpha \nabla_{\theta} L $$
其中 $\nabla L$ 是损失函数对参数的偏导数。
*   **链式法则 (Chain Rule)**：也就是 Backpropagation（反向传播）的数学基础。
    $$ \frac{\partial L}{\partial x} = \frac{\partial L}{\partial y} \cdot \frac{\partial y}{\partial x} $$
    PyTorch 的 `autograd` 引擎本质上就是一个自动执行链式法则求导的计算器。

---

### 总结：代码与数学的映射表

| 代码概念 (Code) | 数学领域 (Math Field) | 核心公式/概念 (Concept) |
| :--- | :--- | :--- |
| `Tensor` (Rank > 0) | **线性代数** | 向量空间、矩阵运算 |
| `nn.Linear` / `matmul` | **线性代数** | 线性映射、仿射变换 |
| `softmax` | **概率论** | 归一化指数函数，分布转换 |
| `Attention` | **几何/统计** | 向量内积（相似度）、加权期望 |
| `1.0 / math.sqrt(d)` | **统计学** | 方差缩放（Variance Scaling） |
| `RoPE` / `sin` / `cos` | **复变函数/三角学** | 欧拉公式、旋转矩阵 |
| `CrossEntropyLoss` | **信息论** | 负对数似然、KL散度 |
| `loss.backward()` | **微积分** | 链式法则 (Chain Rule) |

通过理解这些数学原理，再次阅读Transformer 源码时就明白它们在调用 PyTorch或 TensorFlow的API的背后，是在操纵高维空间中的向量几何，通过概率分布来捕捉语言的本质。