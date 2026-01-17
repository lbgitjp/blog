---
title: 深度学习中的梯度通用公式
description: 通过简单示例解释梯度通用公式及梯度消失梯度爆炸。
slug: dl-03
date: 2026-01-17 00:00:00+0000
categories:
    - 深度学习
    -
tags:
    - 深度学习
    - 反向传播
# 参考：
#
---

假设总层数为 $L_{total}$，并关注从**输出层**到**目标层 $n$** 之间的那段路径。

这是针对任意第 $n$ 层权重 $w_n$ 的梯度通用公式：

## 1. 通用梯度公式

$$ \frac{\partial L}{\partial w_n} = \underbrace{\frac{\partial L}{\partial y}}_{\text{Loss导数}} \cdot \underbrace{\left[ \prod_{k=n+1}^{L_{total}} \left( \sigma'(z_k) \cdot w_k \right) \right]}_{\text{链式连乘项 (从输出层回传到 n+1 层)}} \cdot \underbrace{\sigma'(z_n)}_{\text{当前层激活导数}} \cdot \underbrace{a_{n-1}}_{\text{当前层输入}} $$

## 2. 公式逐项拆解

为了看得更清楚，我们将这个公式拆解为四个部分，分别对应反向传播中的不同角色：

1.  **$\frac{\partial L}{\partial y}$ (Loss Gradient)**:
    *   这是梯度的**源头**。
    *   即损失函数对最终预测值的导数（例如 MSE 中的 $-(Y - \hat{Y})$）。

2.  **$\prod_{k=n+1}^{L_{total}} (\dots)$ (The Chain / Backprop Path)**:
    *   这是**核心连乘项**，也是梯度消失/爆炸的“罪魁祸首”。
    *   它代表了误差信号从最后一层 ($L_{total}$) 一路传导回第 $n+1$ 层所经过的所有变换。
    *   **如果 $n$ 靠近输出层**：这个连乘项很短，梯度通常比较正常。
    *   **如果 $n$ 靠近输入层**：这个连乘项极长，包含大量的 $w$ 和 $\sigma'$ 相乘。

3.  **$\sigma'(z_n)$ (Local Activation Gradient)**:
    *   这是**当前层 $n$** 的激活函数导数。
    *   它决定了误差信号进入该层线性部分时的衰减程度。

4.  **$a_{n-1}$ (Layer Input)**:
    *   这是**当前层 $n$** 的输入值（即上一层 $n-1$ 的输出）。
    *   在计算权重梯度时，必须乘上这个输入值（因为 $z = w \cdot a + b$，对 $w$ 求导就是 $a$）。

---

## 3. 这个公式如何解释“梯度消失”？

通过这个通用公式，我们可以更直观地定义什么时候发生梯度消失。

假设我们需要更新靠近输入层的权重（即 $n$ 很小，而 $L_{total}$ 很大），中间的连乘项会有几十甚至上百项：

$$ \text{连乘项} = (\sigma' \cdot w)_{L_{total}} \cdot (\sigma' \cdot w)_{L_{total}-1} \cdots (\sigma' \cdot w)_{n+1} $$

**梯度消失的数学条件：**
$$ | \sigma'(z_k) \cdot w_k | < 1 $$

如果每一层的这个乘积都小于 1（例如使用 Sigmoid，$\sigma'$ 最大才 0.25，若 $w$ 也是小数值），那么：
$$ 0.25 \times 0.25 \times \dots \times 0.25 \approx 0 $$
导致 $\frac{\partial L}{\partial w_n} \to 0$，深层网络的浅层参数无法更新。

**梯度爆炸的数学条件：**
$$ | \sigma'(z_k) \cdot w_k | > 1 $$

如果权重 $w$ 初始化得很大（例如 $w=2$），且激活函数导数不衰减（如 ReLU 在正区间导数为 1）：
$$ 2 \times 2 \times \dots \times 2 \to \infty $$
导致 $\frac{\partial L}{\partial w_n} \to \text{NaN}$，模型崩溃。

## 4. 下面是**严格对应**通用梯度公式的代码

代码中的变量命名完全照搬了公式中的术语（例如 `chain_product`, `local_sigma_prime`, `layer_input`），并把反向传播拆解得非常细致。

### 核心公式回顾
$$ \frac{\partial L}{\partial w_n} = \underbrace{\frac{\partial L}{\partial y}}_{\text{项1: Loss导数}} \cdot \underbrace{\left[ \prod_{k=n+1}^{L_{total}} \left( \sigma'(z_k) \cdot w_k \right) \right]}_{\text{项2: 链式连乘积 (来自后层)}} \cdot \underbrace{\sigma'(z_n)}_{\text{项3: 当前层导数}} \cdot \underbrace{a_{n-1}}_{\text{项4: 当前层输入}} $$

### 代码实现
请特别留意 `backward` 函数中的注释，每一行都对应公式中的一项。

```python
import numpy as np

class StrictFormulaNN:
    def __init__(self, layer_sizes):
        self.weights = []
        self.biases = []
        self.layer_sizes = layer_sizes
        
        # 初始化权重 (Standard Normal Distribution)
        for i in range(len(layer_sizes) - 1):
            n_in = layer_sizes[i]
            n_out = layer_sizes[i+1]
            self.weights.append(np.random.randn(n_in, n_out) * 0.1)
            self.biases.append(np.zeros((1, n_out)))
            
    def sigmoid(self, x):
        return 1 / (1 + np.exp(-x))

    def sigmoid_derivative(self, z):
        # 注意：公式中写的是 σ'(z)，这里为了严谨，我们直接用 z 计算
        # 虽然用 a * (1-a) 更快，但为了对应公式，我们用 z 算
        s = 1 / (1 + np.exp(-z))
        return s * (1 - s)

    def forward(self, X):
        """
        前向传播：必须缓存 z (线性输出) 和 a (激活输出)
        公式需要用到 z_n 和 a_{n-1}
        """
        self.zs = []      # 存储所有的 z_n
        self.as_ = [X]    # 存储所有的 a (a_0 是输入 X)
        
        input_data = X
        for W, b in zip(self.weights, self.biases):
            z = np.dot(input_data, W) + b
            a = self.sigmoid(z)
            
            self.zs.append(z)
            self.as_.append(a)
            input_data = a
            
        return self.as_[-1]

    def backward(self, y_true, learning_rate=0.1):
        """
        严格对应公式的梯度计算
        """
        m = y_true.shape[0]
        grads_w = [None] * len(self.weights)
        grads_b = [None] * len(self.biases)
        
        # --- [项1] Loss 导数 (dL/dy) ---
        # 假设 Loss = 0.5 * (y - y_true)^2
        # dL/dy = (y_pred - y_true)
        y_pred = self.as_[-1]
        loss_derivative = (y_pred - y_true)
        
        # 初始化 [项2] 链式连乘积 (Chain Product)
        # 在最顶层 (Output Layer) 后面没有其他层了，所以连乘项初始为 Loss导数本身
        # 随着循环进行，这个变量会积累后面所有层的 (σ' * w)
        chain_product = loss_derivative
        
        # 从最后一层 (L-1) 遍历到 第一层 (0)
        num_layers = len(self.weights)
        
        for n in range(num_layers - 1, -1, -1):
            # 获取公式所需的变量
            z_n = self.zs[n]          # 当前层的 z
            # 注意!!!, 由于self.as_的长度比self.zs的长度多1，所以a_prev是self.as_[n]
            a_prev = self.as_[n]      # [项4] 当前层的输入 a_{n-1}
            W_n = self.weights[n]     # 当前层的权重
            
            # --- [项3] 当前层的激活导数 σ'(z_n) ---
            local_sigma_prime = self.sigmoid_derivative(z_n)
            
            # 计算当前层的误差项 delta_n
            # 这里的运算逻辑是：(连乘项) * (当前层导数)
            # 对应公式中的： [LossGrad * Chain] * σ'(z_n)
            delta_n = chain_product * local_sigma_prime
            
            # --- 计算并存储梯度 ---
            # dL/dw_n = [项4: a_{n-1}] * [delta_n]
            # 注意矩阵乘法顺序：Input.T dot Delta
            grads_w[n] = np.dot(a_prev.T, delta_n) / m
            grads_b[n] = np.sum(delta_n, axis=0, keepdims=True) / m
            
            # --- 更新 [项2] 链式连乘积 (为下一层 n-1 做准备) ---
            # 此时我们要穿过当前层，往回传。
            # 根据链式法则，传给下一层的信号需要乘以当前层的权重 W_n
            # 对应公式连乘符号里的乘法操作： * w_k
            chain_product = np.dot(delta_n, W_n.T)

        # --- 应用梯度更新 ---
        for i in range(num_layers):
            self.weights[i] -= learning_rate * grads_w[i]
            self.biases[i]  -= learning_rate * grads_b[i]

# --- 验证代码 ---
# 数据
X = np.array([[0,0], [0,1], [1,0], [1,1]])
Y = np.array([[0], [1], [1], [0]])

# 初始化网络 [2 -> 3 -> 1]
nn = StrictFormulaNN([2, 3, 1])

# 训练一步演示
print("--- Training Step 1 ---")
pred = nn.forward(X)
loss = np.mean(0.5 * (pred - Y)**2)
print(f"Initial Loss: {loss:.6f}")

nn.backward(Y)

pred_new = nn.forward(X)
loss_new = np.mean(0.5 * (pred_new - Y)**2)
print(f"Loss after 1 step: {loss_new:.6f}")
print("Loss decreased:", loss_new < loss)
```

### 代码与公式的映射表

| 代码变量 | 公式项 | 解释 |
| :--- | :--- | :--- |
| `loss_derivative` | $\frac{\partial L}{\partial y}$ | **源头**：损失函数对最终输出的导数。 |
| `chain_product` | $\frac{\partial L}{\partial y} \cdot \prod (\sigma' \cdot w)$ | **传递者**：包含从 Loss 到当前层后面所有层的导数积累。在计算当前层梯度前，它代表“后方传来的误差总和”。 |
| `local_sigma_prime` | $\sigma'(z_n)$ | **门控**：当前层的激活函数导数，控制误差能保留多少穿过激活层。 |
| `delta_n` | 上述三项的乘积 | **总误差**：当前层线性输出 $Z_n$ 的敏感度。 |
| `a_prev` | $a_{n-1}$ | **输入**：当前层的输入值，根据 $z=wa+b$，对 $w$ 求导就是 $a$。 |

### 关键点解析：代码中的递归体现

你可能注意到代码里没有显式的写一个 $\prod$ (连乘符号)。这是因为编程中的 `chain_product = np.dot(delta_n, W_n.T)` 这一行赋值语句，**在循环过程中自然地完成了连乘**。

1.  **初始状态 ($n=Output$)**：`chain_product` 只有 $\frac{\partial L}{\partial y}$。
2.  **循环第一轮结束**：`chain_product` 变成了 $\frac{\partial L}{\partial y} \cdot \sigma'(z_{out}) \cdot W_{out}$。
3.  **循环第二轮结束**：`chain_product` 变成了 $\frac{\partial L}{\partial y} \cdot [\sigma'(z_{out}) \cdot W_{out}] \cdot [\sigma'(z_{hidden}) \cdot W_{hidden}]$。

这正是公式中那个连乘符号 $\prod$ 的严格代码实现。

## 5. 代码视角：PyTorch 中的 `grad_output`

如果写过 PyTorch 的自定义 Autograd Function，会看到 `backward` 函数的参数里有一个 `grad_output`。

```python
class MyLinear(torch.autograd.Function):
    @staticmethod
    def forward(ctx, input, weight):
        ctx.save_for_backward(input, weight)
        return input @ weight.t()

    @staticmethod
    def backward(ctx, grad_output):
        # 这个grad_output对应前面代码中的chain_product
        input, weight = ctx.saved_tensors

        # 对应前面公式连乘符号里的乘法操作: * w_k
        grad_input = grad_output @ weight

        # 对应前面公式中的项4: 当前层输入: * a_{n-1}
        grad_weight = grad_output.t() @ input

        return grad_input, grad_weight
```

*   `grad_output`：这就是从上一层（逻辑上的后一层）传回来的chain_product。
*   `backward` 的任务：利用这个chain_product，计算出当前层输入的梯度 `grad_input`（即传给下一层的 chain_product）。
