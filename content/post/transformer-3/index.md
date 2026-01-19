---
title: RoPE(旋转位置编码)
description: 这篇文章旨在介绍现代LLM 的位置编码标准——RoPE (Rotary Positional Embeddings)。
slug: transformer-3
date: 2026-01-19 00:00:00+0000
categories:
    - transformer
    -
tags:
    - 大语言模型
    - LLM
    - RoPE
# 参考：
# https://aistudio.google.com/app/u/2/prompts/1sGWRSOvHvRzBxMNekFKwKKvUNON-pFCT
---

# 深度解构：现代 LLM 的位置编码标准 —— RoPE (Rotary Positional Embeddings)

在 Transformer 架构的进化史中，位置编码经历了从 **绝对位置相加 (Sinusoidal/Learnable)** 到 **相对位置偏置 (ALiBi)** 的演变。而最终统治当前大模型（Llama 3, GPT-4, PaLM）的是 **RoPE (旋转位置编码)**。

RoPE 的核心洞见在于：**通过将向量在每一个维度对中进行旋转，使得两个向量的点积仅取决于它们的相对距离，而非绝对位置。**

本文将揭示其在生产环境中的真实实现形态。

---

## 1. 核心数学：从 2D 到 ND

### 1.1 直观理解：二维平面上的旋转
在二维平面上，将向量 $\mathbf{x}$ 旋转 $\theta$ 角度的线性变换为：
$$
\begin{pmatrix} x' \\ y' \end{pmatrix} = \begin{pmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{pmatrix} \begin{pmatrix} x \\ y \end{pmatrix}
$$

为了理解 RoPE，我们先只看一个 **二维向量** $(x_1, x_2)$。

假设这个向量出现在序列的第 $m$ 个位置。
RoPE 的做法是：**把这个向量在二维平面上逆时针旋转 $m \cdot \theta$ 角度。**

在线性代数中，旋转一个向量是由**旋转矩阵**完成的：

$$
\mathbf{R}_{\theta, m} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} =
\begin{pmatrix}
\cos(m\theta) & -\sin(m\theta) \\
\sin(m\theta) & \cos(m\theta)
\end{pmatrix}
\begin{pmatrix} x_1 \\ x_2 \end{pmatrix}
$$

展开计算后，新的向量 $(x'_1, x'_2)$ 变成：

$$
\begin{aligned}
x'_1 &= x_1 \cos(m\theta) - x_2 \sin(m\theta) \\
x'_2 &= x_1 \sin(m\theta) + x_2 \cos(m\theta)
\end{aligned}
$$

这就是 RoPE 的基本原子操作。

### 1.2 相对位置的魔法, 为什么旋转能带来“相对位置”？
假设 Query 向量在位置 $m$，Key 向量在位置 $n$。
RoPE 将它们分别旋转 $m\theta$ 和 $n\theta$。
当计算 Attention Score ($Q \cdot K^T$) 时，根据复数运算法则（或三角恒等式）：
$$
\text{Score} \propto \text{Real}( (q e^{im\theta}) \cdot (k e^{in\theta})^* ) = \text{Real}( q k^* e^{i(m-n)\theta} )
$$
**结果只包含 $(m-n)$，即相对位置。** 这赋予了模型极强的长文本外推能力（Extrapolation）。

我们来看看 Attention 的核心操作——**点积 (Dot Product)**。

假设：
*   **Query ($q$)** 在位置 $m$，旋转了 $m\theta$。
*   **Key ($k$)** 在位置 $n$，旋转了 $n\theta$。

它们的点积计算如下（利用复数或三角恒等式）：
$$
\langle f(q, m), f(k, n) \rangle = \text{Real}( (q e^{im\theta}) \cdot (k e^{in\theta})^* )
$$
$$
= q^T \mathbf{R}_m^T \mathbf{R}_n k
$$

根据旋转矩阵的性质：**先旋转 $n$ 度，再反向旋转 $m$ 度（转置即逆旋转），等于旋转了 $n-m$ 度。**
$$
\mathbf{R}_m^T \mathbf{R}_n = \mathbf{R}_{n-m}
$$

所以，点积的结果只与 $(n-m)$ 有关：
$$
\text{Attention Score} \propto \cos((n-m)\theta)
$$

**结论**：虽然我们给 $q$ 和 $k$ 注入的是**绝对位置** ($m$ 和 $n$)，但在做注意力运算时，模型感知到的完全是**相对距离** ($n-m$)。这使得模型具有极强的外推能力（比如训练时只见过长度 2000，推理时能处理 4000）。

### 1.3 角度计算公式：为什么是 $base^{-2i/d}$？

RoPE 的目标是给每一个维度赋予一个**旋转频率（角速度）**。

*   **设计原则**：借鉴傅里叶变换或原始 Transformer 的正弦编码，我们希望：
    *   **低维度（$i=0$）**：转得快（高频），捕捉短距离关系。
    *   **高维度（$i=d/2$）**：转得慢（低频），捕捉长距离关系。

**公式定义**：
对于第 $i$ 组（$i \in [0, d/2)$），其旋转频率 $\theta_i$ 为：
$$ \theta_i = \text{base}^{-2i / d} $$

*   **当 $i=0$ (第一个维度)**:
    $$ \theta_0 = \text{base}^{-0/d} = \text{base}^0 = 1 $$
    这意味着位置每增加 1，它旋转 1 弧度。这是**最高频**。
*   **当 $i \to d/2$ (最后一个维度)**:
    $$ \theta \to \text{base}^{-1} = 1/\text{base} $$
    如果 base=10000，则位置每增加 1，它只旋转 1/10000 弧度。这是**最低频**。

---

## 2. 生产环境的工程优化：Half-Split 模式

这是教科书与 PyTorch 源码最大的区别。

*   **理论上的 RoPE**: 将向量相邻元素两两分组 $(x_0, x_1), (x_2, x_3)...$ 进行旋转。
*   **实际的 RoPE (Llama/HuggingFace)**: 采用 **Half-Split** 策略。
    将向量一分为二，让 $x_i$ 与 $x_{i + D/2}$ 配对。

**为什么？**
为了 **GPU 访存效率**。在显存中连续读取一块数据（`chunk`）远比跨步读取（`strided slice` 如 `x[::2]`）要快得多。

对应的数学操作变化：
$$
\text{rotate\_half}(\mathbf{x}) = \text{cat}(-\mathbf{x}_{后半}, \mathbf{x}_{前半})
$$
配合的频率表 $\cos, \sin$ 也需要构造为：$[\cos\theta, \cos\theta]$ 的形式。

下面举例说明, 假设向量维度 $D=1024$。

### 2.1. 定义组件

首先，我们将输入向量 $\mathbf{x}$ 一分为二：
$$
\mathbf{x} = [\mathbf{x}_a, \mathbf{x}_b]
$$
其中：
*   $\mathbf{x}_a = \mathbf{x}[0 : 512]$ （前 512 维）
*   $\mathbf{x}_b = \mathbf{x}[512 : 1024]$ （后 512 维）

接着，定义频率向量 $\boldsymbol{\theta}$（共 512 个角度）：
$$
\boldsymbol{\theta} = [\theta_0, \theta_1, \dots, \theta_{511}]
$$

为了进行向量化计算，我们需要构造**全尺寸**的 $\cos$ 和 $\sin$ 向量。根据代码逻辑，它们是**重复拼接**的：
$$
\mathbf{C} = [\cos\boldsymbol{\theta}, \cos\boldsymbol{\theta}] \in \mathbb{R}^{1024}
$$
$$
\mathbf{S} = [\sin\boldsymbol{\theta}, \sin\boldsymbol{\theta}] \in \mathbb{R}^{1024}
$$

### 2.2. 定义 rotate_half 操作

`rotate_half(x)` 的操作是将后半部分变负并移到前面，将前半部分移到后面：
$$
\text{rotate\_half}(\mathbf{x}) = [-\mathbf{x}_b, \mathbf{x}_a]
$$
展开来看就是：
$$
[-\mathbf{x}[512:1024], \quad \mathbf{x}[0:512]]
$$

### 2.3. 最终的全局向量公式

现在，我们可以写出完整的 RoPE 向量计算公式。

**总公式：**
$$
\text{RoPE}(\mathbf{x}) = \mathbf{x} \odot \mathbf{C} + \text{rotate\_half}(\mathbf{x}) \odot \mathbf{S}
$$

---

### 2.4. 拆解验证（看看每一半发生了什么）

把上面的公式代入 $\mathbf{x}_a$ 和 $\mathbf{x}_b$ 进行拆解，你会发现它完美对应了旋转逻辑：

$$
\begin{aligned}
\text{RoPE}(\mathbf{x}) &= [\mathbf{x}_a, \mathbf{x}_b] \odot [\cos\boldsymbol{\theta}, \cos\boldsymbol{\theta}] + [-\mathbf{x}_b, \mathbf{x}_a] \odot [\sin\boldsymbol{\theta}, \sin\boldsymbol{\theta}] \\
&= \underbrace{[\mathbf{x}_a \odot \cos\boldsymbol{\theta}, \quad \mathbf{x}_b \odot \cos\boldsymbol{\theta}]}_{\text{第一项}} + \underbrace{[-\mathbf{x}_b \odot \sin\boldsymbol{\theta}, \quad \mathbf{x}_a \odot \sin\boldsymbol{\theta}]}_{\text{第二项}}
\end{aligned}
$$

**最后合并结果（按位置分块）：**

*   **前半部分 (0~511)**:
    $$ \text{Result}_{前半} = \mathbf{x}_a \odot \cos\boldsymbol{\theta} - \mathbf{x}_b \odot \sin\boldsymbol{\theta} $$
    *(这正是旋转公式的实部：$x \cos - y \sin$)*

*   **后半部分 (512~1023)**:
    $$ \text{Result}_{后半} = \mathbf{x}_b \odot \cos\boldsymbol{\theta} + \mathbf{x}_a \odot \sin\boldsymbol{\theta} $$
    *(这正是旋转公式的虚部：$y \cos + x \sin$)*

### 总结

这就是现代 LLM 中 RoPE 的终极形态：**利用向量拼接和翻转，一次性对 1024 维向量完成 512 组独立的 2D 旋转。**

## 3. 代码实现 (PyTorch)

主要分为两步：
1.  **`calc_rope_params`**: 计算出 $\cos$ 和 $\sin$ 表（对应角度计算公式）。
2.  **`apply_rope`**: 执行旋转（对应旋转公式）。

```python
import torch

def calc_rope_params(dim: int, seq_len: int, base: int = 10000):
    """
    第一步：计算 Cos 和 Sin 表
    对应数学公式：theta_i = base ^ (-2i / dim)
    
    Args:
        dim: Head Dimension (e.g., 64)
        seq_len: 当前序列长度 (e.g., 10)
        base: 旋转基数
    Returns:
        cos, sin: 形状为 [seq_len, dim]
    """
    # -------------------------------------------------------------
    # 1. 计算角速度 (Inverse Frequencies)
    # -------------------------------------------------------------
    # 生成偶数索引: [0, 2, 4, ..., dim-2] -> Shape: [dim/2]
    # 对应公式中的 2i
    indices = torch.arange(0, dim, 2).float()
    
    # 计算 theta = base ^ (-2i / d)
    # 这一步对应了上面数学部分的公式 1
    theta = 1.0 / (base ** (indices / dim))  # Shape: [dim/2]
    
    # -------------------------------------------------------------
    # 2. 计算每个位置的绝对旋转角度 (Position * Theta)
    # -------------------------------------------------------------
    # 生成位置序列: [0, 1, ..., seq_len-1] -> Shape: [seq_len]
    pos_idx = torch.arange(seq_len).float()
    
    # 外积计算: pos * theta
    # idx[i, j] 表示第 i 个位置，第 j 组向量需要的旋转角度
    # Shape: [seq_len, dim/2]
    idx_theta = torch.outer(pos_idx, theta)
    
    # -------------------------------------------------------------
    # 3. 拼接频率 (Half-Split 模式)
    # -------------------------------------------------------------
    # 我们需要构建 [cos(theta), cos(theta)] 的形式
    # 这样前一半 dim 和 后一半 dim 使用相同的频率
    # Shape: [seq_len, dim]
    freqs = torch.cat((idx_theta, idx_theta), dim=-1)
    
    # 计算 cos 和 sin
    return freqs.cos(), freqs.sin()


def rotate_half(x: torch.Tensor):
    """
    核心辅助函数：实现 [-x2, x1] 的变换
    """
    # 将向量 x 从中间切开
    # x1 shape: [..., dim/2]
    # x2 shape: [..., dim/2]
    x1, x2 = x.chunk(2, dim=-1)
    
    # 拼按接为 [-x2, x1]
    # Shape: [..., dim]
    return torch.cat((-x2, x1), dim=-1)


def apply_rope(x: torch.Tensor, cos: torch.Tensor, sin: torch.Tensor):
    """
    第二步：执行旋转
    对应数学公式：x' = x*cos - y*sin
    
    Args:
        x: 输入 Query 或 Key, Shape [B, H, L, D]
        cos: 预计算好的余弦表, Shape [L, D] (需要广播)
        sin: 预计算好的正弦表, Shape [L, D] (需要广播)
    """
    # -------------------------------------------------------------
    # 1. 调整 cos/sin 形状以支持广播
    # -------------------------------------------------------------
    # 输入 cos: [L, D] -> 变成 [1, 1, L, D]
    # 这样它可以自动适配 Batch(B) 和 Heads(H)
    cos = cos[None, None, :, :]
    sin = sin[None, None, :, :]
    
    # -------------------------------------------------------------
    # 2. 核心旋转公式 (Core Formula)
    # -------------------------------------------------------------
    # 这一步完全对应上面数学部分的公式 2
    # x * cos           -> [x1*cos, x2*cos]
    # rotate_half(x)    -> [-x2,    x1]
    # ... * sin         -> [-x2*sin, x1*sin]
    # 相加               -> [x1*cos - x2*sin, x2*cos + x1*sin]
    #
    # Shape 变化: [B, H, L, D] (始终保持不变)
    return (x * cos) + (rotate_half(x) * sin)

# ==========================================
# 极简运行示例
# ==========================================
if __name__ == "__main__":
    # 配置
    dim = 8        # 假设维度为 8
    seq_len = 3    # 假设序列长度为 3
    
    # 1. 计算参数
    # cos, sin shape: [3, 8]
    cos, sin = calc_rope_params(dim, seq_len)
    
    # 2. 模拟输入 x
    # [Batch=1, Heads=1, L=3, D=8]
    x = torch.randn(1, 1, seq_len, dim)
    
    # 3. 应用 RoPE
    x_rotated = apply_rope(x, cos, sin)
    
    print(f"Input shape: {x.shape}")
    print(f"Cos shape:   {cos.shape}")
    print(f"Output shape:{x_rotated.shape}")
    
    # 验证第一个位置 (pos=0) 是否没有旋转 (cos=1, sin=0)
    # 因为 pos=0 时，旋转角度为 0
    # 所以 x_rotated[0,0,0] 应该等于 x[0,0,0]
    print(f"\nPos 0 difference: {(x_rotated[:,:,0,:] - x[:,:,0,:]).abs().sum().item()}")
```

### 总结

1.  **关于公式**：`base^(-2i/d)` 保证了不同维度有从快到慢的旋转速度。
2.  **关于计算**：`(x * cos) + (rotate_half(x) * sin)` 是一种利用向量加减乘法来**模拟复数旋转（矩阵乘法）**的高效数学技巧。
3.  **关于代码**：上面的代码展示了 RoPE 最本质的逻辑。实际 Llama 模型中只是把 `calc_rope_params` 的结果缓存了起来，计算逻辑完全一致。