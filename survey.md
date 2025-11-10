---
bibliography: ["humanizer.bib"] 
---

# 1. 研究概况

[@mengGradEscapeGradientBasedEvader2025]对AI Detector和AI Humanizer的源流和分类做了系统的综述，概要如下：

## Detector分类

Detector分为事后（post-hoc）和主动（proactive）两类。

### 事后检测器

事后检测器又分为深度学习检测器和统计检测器。

**深度学习检测器**

- **定义**：通过微调语言模型（LM）得到的二分类器
- **特点**：简单有效，但需要大量训练数据
- **代表方法**：
    - CheckGPT（冻结LM，增加BiLSTM头）
    - MPU（多尺度模块和PU损失，提升短文本检测）
    - DEMASQ（基于能量模型）
- **优势**：准确率高
- **劣势**：需大数据

**统计检测器**

- **代表方法**：
    - GLTR（跟踪每个token概率）
    - DetectGPT（扰动文本后比较困惑度变化）
- **优势**：适合小样本场景

### 主动检测器

需要修改大模型推理过程，通常由服务商维护并提供API

**水印法**

- **原理**：在生成时嵌入不可见水印，检测时测量水印强度
- **代表方法**：KGW方法
- **优点**：检测原理稳健
- **缺点**：影响文本质量，易被同义改写攻击破解

**检索法**

- **原理**：将历史生成文本存库，检测时比对
- **缺点**：涉及用户数据存储，存在隐私和合规风险，存储量巨大

## Humanizer分类

Humanizer分为基于扰动（perturbation-based）和基于释义（paraphrase-based）两类。

### 基于扰动（Perturbation-based）

- **特点**：通过替换文本中的特定词语来逃避检测
- **缺点**：会降低文本流畅性和可读性
- **代表方法**：
  - RP（用同义词随机替换词语）
  - DFTFooler（利用AIGT困惑度低的特点，替换语言模型最有信心的前N个词）
  - TextFooler（通过查询检测器找出重要词并替换）

### 基于释义（Paraphrase-based）

- **特点**：通过释义重写文本，改变表达但不改变含义
- **缺点**：难以精确控制修改幅度，调节选项有限
- **代表方法**：
  - DIPPER（大规模seq2seq模型，能控制词汇和顺序多样性）
  - SentPara（逐句释义后重组，模型参数远小于DIPPER）

# 2. 工作归纳

文献调研发现最近比较好的工作都是基于**微调语言模型的Detector**和**基于释义的Humanizer**，下面分别介绍。其中开源的方法大都不适用，最新的且比较好的适用的方法都需要自己根据论文来写算法。

## 2.1 自定义损失函数的对抗方法

**论文**：[@mengGradEscapeGradientBasedEvader2025]（开源）

### 2.1.1 模型配置

- **模型大小**：T5-139M
- **硬件配置**：2 × A100-80GB GPU节点，32个CPU核心，1TB内存

### 2.1.2 数据集

- GROVER News
- HC3
- GPA
- GPTWiki Intro

### 2.1.3 评测的检测器

**微调模型：**
- RoBERTa
- GPT2
- BERT

**专用检测器：**
- MPU
- CheckGPT

**统计方法：**
- GLTR

### 2.1.4 基准Humanizer

**基于扰动：**
- RP
- DFTFooler

**基于释义：**
- DIPPER
- SentPara

### 2.1.5 评价指标

**逃避效果：**
- Evasion Rate

**文本质量：**
- ROUGE
- Cos-sim
- Perplexity
- GRUEN

### 2.1.6 核心方法

用以下损失函数训练一个深度学习Paraphraser:

$$\min \quad \mathcal{L}_{D_V} \left( \mathcal{F}_\theta \left( \{x_i\}_{i=1}^n \right), y_{\text{human}} \right)$$

$$\text{s.t.} \quad \text{dis}_{\text{syn}} \left( \mathcal{F}_\theta \left( \{x_i\}_{i=1}^n \right), \{x_i\}_{i=1}^n \right) < T_{\text{syn}}$$

$$\text{dis}_{\text{sem}} \left( \mathcal{F}_\theta \left( \{x_i\}_{i=1}^n \right), \{x_i\}_{i=1}^n \right) < T_{\text{sem}}$$

**符号说明：**

- $\mathcal{F}_\theta$：最终要训练得到的Paraphraser
- $\mathcal{L}_{D_V}$：检测器$\mathcal{D_V}$的交叉熵损失
- $\text{dis}_{\text{syn}}$和$\text{dis}_{\text{sem}}$：分别测量输入和输出之间的句法距离和语义距离
- $T_{\text{syn}}$和$T_{\text{sem}}$：分别表示句法阈值和语义阈值

**约束作用：**

- $T_{\text{syn}}$：限制$\mathcal{F}_\theta$可以改变文本结构的程度
- $T_{\text{sem}}$：确保$\mathcal{F}_\theta$不会显著改变文本的含义

### 2.1.7 优缺点分析

**✓ 优点：**

- 不需要提供人类文本，直接用AI文本即可

**✗ 挑战：**

- 需要$\mathcal{L}_{D_V}$是开源的
- 对于黑盒模型需要使用模型窃取技术探查得到Tokenizer和推断模型结果，然后重新训练替代模型，这本身就很困难

## 2.2 强化学习方法（DPO）

**论文**：[@nicksLanguageModelDetectors2023]（开源）

### 2.2.1 模型配置

- **模型大小**：7B parameter Llama-2-base model

### 2.2.2 数据集

- OpenWebText
- Alpaca instruction dataset
- 人写的文章标题和AI生成的文章内容
- Writing Prompts dataset
- 使用Llama-2-7B生成的样本构建偏好数据集

### 2.2.3 评测的检测器

**微调模型：**
- RoBERTa-large
- RoBERTa-base

**Zero-shot统计：**
- Log Rank
- Log Probability

**Zero-shot扰动：**
- DetectGPT
- DetectLLM

**商业检测器：**
- GPTZero
- Sapling
- Originality.ai (model 2.0)
- Winston AI

### 2.2.4 基准Humanizer

N/A

### 2.2.5 评价指标

**检测性能：**
- AUROC

**文本质量：**
- Perplexity
- GPT4 win rate
- Offline entropy

### 2.2.6 核心方法

使用DPO对Llama-2-7B进行微调

### 2.2.7 优缺点分析

**✗ 缺点：**

- 本质上是用有监督分类方法训练一个Detector

## 2.3 有监督微调方法

**论文**：[@krishnaParaphrasingEvadesDetectors2023]（开源）

### 2.3.1 模型配置

- **模型大小**：T5-XXL 11B

### 2.3.2 数据集

**训练数据：**
- PAR3 dataset构建对齐数据

**测试数据：**
- GPT2-XL (1.5B) 生成300 tokens
- OPT-13B 生成300 tokens
- text-davinci-003 生成300 tokens
- WikiText-103验证集3K个两句话块+接下来300 tokens（人类文本）

### 2.3.3 评测的检测器

**水印检测：**
- Watermarking

**统计方法：**
- DetectGPT

**商业检测器：**
- GPTZero
- OpenAI's text classifier

**排序方法：**
- RankGenXL-all

### 2.3.4 基准Humanizer

N/A

### 2.3.5 评价指标

**检测性能：**
- Detection Accuracy

**语义保持：**
- Semantic similarity

### 2.3.6 核心方法

- 人工构造对齐数据
- 使用SFT方式让模型学会如何paraphrase

## 2.4 强化学习方法（RADAR）

**论文**：[@huRADARRobustAIText2023]（闭源）

### 2.4.1 模型配置

- **模型**：T5-large和RoBERTa-large分别作为paraphraser和detector的初始化
- **硬件配置**：2 GPUs (NVIDIA Tesla V100 32GB)

### 2.4.2 数据集

**训练数据：**
- WebText采样160K文档（人类文本）

**测试数据（人类）：**
- Xsum
- SQuAD
- Reddit WritingPrompts (WP)
- TOEFL

**测试数据（AI模型）：**
- Pythia-2.8B
- Dolly-V2-3B
- Palmyra-base
- Camel-5B
- GPT-J-6B
- Dolly-V1-6B
- LLaMA-7B
- Vicuna-7B

### 2.4.3 评测的检测器

**微调模型：**
- OpenAI (RoBERTa) model

**统计方法：**
- Log probability
- Rank
- Log rank
- Entropy
- DetectGPT

### 2.4.4 基准Humanizer

N/A

### 2.4.5 评价指标

**检测性能：**
- AUROC

**文本质量：**
- GPT-3.5-Turbo评分
- Quora Question Pairs
- iBLEU

### 2.4.6 核心方法

训练流程包括以下步骤：

1. 使用要攻击的AI模型对人类文本做生成形成AI文本
2. 使用paraphraser对AI文本做转移形成转义文本
3. 使用detector对转义文本打分
4. 使用PPO和打分更新phraser
5. 使用人类文本、AI文本和转义文本更新detector

### 2.4.7 优缺点分析

**✗ 缺点：**

- 官方没有release训练代码和数据，只有detector的checkpoint

# 参考文献
