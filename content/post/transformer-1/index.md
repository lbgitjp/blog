---
title: 注意力可视化 (Attention Visualization)的工具或方法
description: 注意力可视化是 “可解释性 AI (Explainable AI / XAI)” 和 “机械可解释性 (Mechanistic Interpretability)” 领域的核心内容。
slug: transformer-1
date: 2026-01-10 06:00:00+0000
categories:
    - transformer
    -
tags:
    - 大语言模型
    - LLM
    - 注意力机制
    - Attention
# 参考：
#
---

注意力可视化是 **“可解释性 AI (Explainable AI / XAI)”** 和 **“机械可解释性 (Mechanistic Interpretability)”** 领域的核心内容。

需要明确的是：**只有对于“开源模型”（如 Llama, Mistral, BERT, GPT-2 等），我们才能直接访问其内部权重来绘制注意力图。** 对于 ChatGPT、Claude 等闭源模型，我们无法直接获取注意力权重，只能通过“归因分析”等间接手段来模拟。

以下是目前主流的工具和方法，按从“交互式演示”到“硬核代码库”分类：

### 一、 交互式可视化工具（适合理解原理/教学）
如果你不想写代码，只想看懂注意力机制是如何工作的，这些网页工具是最好的选择。

1.  **Transformer Explainer (Politeknik)**
    *   **特点**：这是一个非常惊艳的网页应用，直接在浏览器中运行轻量级 GPT-2 模型。你可以输入文本，实时看到 Temperature 如何影响选择，以及**模型在生成下一个词时关注了前面的哪些词**。
    *   **适用**：直观感受“为什么模型选了这个词”。
    *   **关键词搜索**：`Transformer Explainer Georgia Tech`
    *   **Github**：[Transformer Explainer](https://github.com/poloclub/transformer-explainer)。

2.  **LLM Visualization (by Brendan Bycroft)**
    *   **特点**：这是一个 3D 可视化项目。它将整个 Transformer 架构（包括每一层的 Attention Head）展开成三维结构。你可以点击具体的某个 Token，看到连线（Attention 权重）连接到了前面的哪些 Token。
    *   **核心功能**：展示了 Query, Key, Value 矩阵是如何计算得出注意力的。
    *   **关键词搜索**：`llm visualization bbycroft`
    *   **Github**：[LLM Visualization](https://github.com/bbycroft/llm-viz)。

### 二、 Python 库（适合开发者/研究人员）
如果你会用 Python 并且在本地运行模型（如 Hugging Face Transformers），这些库是标准配置。

1.  **BERTViz** (经典老牌)
    *   **简介**：专门用于 Transformer 模型注意力可视化的 Jupyter Notebook 工具。
    *   **主要视图**：
        *   *Head View*：展示某一层的某个注意力头（Head）关注了哪些词。
        *   *Model View*：宏观展示所有层和头的注意力模式。
    *   **支持**：BERT, GPT-2, Llama 等绝大多数 Hugging Face 模型。
    *   **代码示例**：
        ```python
        from bertviz import head_view
        head_view(model_attention, tokens)
        ```

2.  **Ecco** (推荐)
    *   **简介**：比 BERTViz 更现代，专为生成式语言模型（Generative LM）设计。
    *   **杀手级功能**：
        *   **Input Saliency**：通过颜色深浅，直接显示生成结果受输入中哪个词的影响最大。
        *   **Neuron Activation**：甚至能探索是哪些神经元被激活了。
    *   **适用**：分析“为什么模型生成了这句话”。

3.  **Inseq** (Interpretability for Sequence Generation)
    *   **简介**：一个较新的库，集成了多种归因方法（Attribution Methods）。不仅有 Attention，还有 Gradient（梯度）分析。
    *   **特点**：可以生成非常漂亮的 HTML 报告，展示输入 Token 对输出 Token 的贡献度（Feature Attribution）。

### 三、 针对闭源模型（ChatGPT/Claude）的“黑盒”方法
由于我们拿不到 OpenAI 的权重，无法画出真正的 Attention Map，但我们可以用**“特征归因” (Feature Attribution)** 来模拟。

1.  **SHAP (SHapley Additive exPlanations)**
    *   **原理**：博弈论方法。它通过不断地“遮盖”输入中的某些词，观察输出概率的变化。
    *   **逻辑**：如果我把“不”字删掉，模型输出“好”的概率从 0.1 变成了 0.9，那么“不”这个词的权重（注意力/重要性）就极高。
    *   **缺点**：计算量大（因为要反复调用 API），且非常费钱。

2.  **LIME (Local Interpretable Model-agnostic Explanations)**
    *   **原理**：与 SHAP 类似，通过扰动输入数据来训练一个简单的线性模型，从而近似解释复杂模型的局部行为。

3.  **OpenAI/Anthropic 的 Logprobs（对数概率）**
    *   虽然不是可视化，但在 API 返回中开启 `logprobs`，可以看到模型在选词时的“自信度”。这可以作为注意力的一个侧面印证（如果概率极高，通常意味着注意力高度集中在某些特定指令或模式上）。

### 四、 应该如何选择？

1.  **我想学习原理，看个热闹**：
    *   去玩 **Transformer Explainer** 或 **LLM Visualization (bbycroft)**。

2.  **我在做本地模型开发 (Llama/Qwen)，想调试 Prompt 或微调效果**：
    *   使用 **BERTViz** 查看注意力头是否“散焦”。
    *   使用 **Ecco** 查看输入词对输出的影响。

3.  **我是 Prompt 工程师，用的是 ChatGPT**：
    *   你无法使用上述可视化工具。
    *   **替代方案**：使用我之前提到的**“剥洋葱法”**（手动删减词语看效果变化），这是最原始但最有效的“人肉注意力分析”。

### 一个简单的代码概念验证 (使用 Hugging Face)
如果你想自己动手写最基础的，Hugging Face 原生支持输出注意力：

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

inputs = tokenizer("Hello world", return_tensors="pt")

# 关键参数：output_attentions=True
outputs = model(**inputs, output_attentions=True)

# 拿到注意力张量：(层数, batch, heads, 序列长度, 序列长度)
# 这里面存的就是所有的“连线权重”
attention_maps = outputs.attentions 
print(attention_maps[-1].shape) 
```

拿到这个 `attention_maps` 矩阵后，你可以用 Matplotlib 的 `imshow` 画一个热力图，那就是最基础的注意力可视化了。