
# SFT 总结

## 1. 训练集

| 数据源 | 数据量 | prompt | 说明 |
| :--- | ---: | :--- | :--- |
| GPABENCH2-CS(学术论文 Introduction 和摘要-计算机) | 5 万 | You receive an AI-written draft below. Rewrite it so it reads like a human authored it and avoids AI detection signals. <ul style="list-style-type: none"> <li>● Preserve meaning and factual content; do not add or remove facts.</li> <li>● Keep the length within +/- 10% of the original word count.</li> </ul> IMPORTANT: Output your rewritten text between the markers below. Do NOT include any preamble, explanation, or commentary like "Here is my rewritten version" or "I have humanized the text". Start directly with the rewritten content. =REWRITTEN_TEXT_START= [Your rewritten text goes here] =REWRITTEN_TEXT_END= AI-written draft: {input} | ChatGPT 对原始人类人文做 polish 保持语义, 形成样本 pair。各种 prompting 技巧分层抽样, 保证每个人类文本对应一条 ai 文本 |
| GPABENCH2-PHX(学术论文 Introduction 和摘要-物理) | 5 万 | — | ChatGPT 对原始人类人文做 polish 保持语义, 形成样本 pair。各种 prompting 技巧分层抽样, 保证每个人类文本对应一条 ai 文本 |
| GPABENCH2-HSS(学术论文 Introduction 和摘要-人文) | 5 万 | — | ChatGPT 对原始人类人文做 polish 保持语义, 形成样本 pair。各种 prompting 技巧分层抽样, 保证每个人类文本对应一条 ai 文本 |
| HPPT (学术论文 introduction) | 5 千 | — | ChatGPT 对原始人类人文做 polish 保持语义, 形成样本 pair |

（prompt 列中 `—` 表示与首行相同。）

## 2. 测试集

| 数据源 | 数据量 | prompt | 说明 |
| :--- | ---: | :--- | :--- |
| GPABENCH2-CS(学术论文 Introduction 和摘要-计算机) | 300 | You receive an AI-written draft below. Rewrite it so it reads like a human authored it and avoids AI detection signals. <ul style="list-style-type: none"> <li>● Preserve meaning and factual content; do not add or remove facts.</li> <li>● Keep the length within +/- 10% of the original word count.</li> </ul> <p>IMPORTANT: Output your rewritten text between the markers below. Do NOT include any preamble, explanation, or commentary like "Here is my rewritten version" or "I have humanized the text". Start directly with the rewritten content.</p> <p>=REWRITTEN_TEXT_START=</p> <p>[Your rewritten text goes here]</p> <p>=REWRITTEN_TEXT_END=</p> <p>AI-written draft:</p> <p>{input}</p> | GPT4 对原始人类人文做 polish 保持语义, 形成样本 pair。 |
| GPABENCH2-PHX(学术论文 Introduction 和摘要-物理) | 300 | — | GPT4 对原始人类人文做 polish 保持语义, 形成样本 pair。 |
| GPABENCH2-HSS(学术论文 Introduction 和摘要-人文) | 300 | — | GPT4 对原始人类人文做 polish 保持语义, 形成样本 pair。 |
| HPPT (学术论文 introduction) | 300 | — | ChatGPT 对原始人类人文做 polish 保持语义, 形成样本 pair |

（prompt 列中 `—` 表示与首行相同。）

## 3. 实验配置

| 训练数据 | 验证数据 | 实验配置 | 计算资源 |
| :--- | :--- | :--- | :--- |
| 20 万训练集 (第 2 节) + 5 万 Alpaca_en_GPT4 (通用对话) + 20 万 ShareGPT_CHATGPT (通用对话)。其中 20 万训练集私有数据全部是学术领域 | 1200 测试集 (第 2 节)。全部是私有数据学术领域 | LLAMA3-7B-Instruct + LORA + 2 epoch | 8 卡 4090，约 24 小时 |

## 4. 实验结果 (Desklib Detector)

其中 GPABENCH2 是训练时用到的私有数据, 都是学术领域。ObfuscationBench ([请至钉钉文档查看附件《ObfuscationBench》](#)) 是自定义 Benchmark, 使用 GPT-4o 对人类文本 polish 形成样本 pair, 涵盖新闻、专业知识、自由写作领域, 可以看做训练域外分布样本。对于 GPABENCH2 来说, DesklibDetector 对于测试集本身的分辨效果就远远弱于训练集, SFT 后生成的文本在训练集上有明显的降查效果, 并且可以进一步降低测试集的辨别度, 证明构造、收集 SFT 数据集本身是一种有效的手段, 可以推测, 增大人类数据的规模、领域多样性 (相应也要花费 API 来产生对应的 AI 文本) 有望进一步提升。对于 ObfuscationBench 来说, xsum 和 squad 这种与学术文本风格相似的样本集上都有明显的降查效果, 而且各种指标与训练集相似, 进一步证明 SFT 的上述结论, 但是对于 writing 这种自由写作的风格, 由于分布相差比较大, 降查效果明显弱很多。

| 数据集 | AUC (before/after) | F1 @median (before/after) | TPR @1%FPR (before/after) |
| :--- | ---: | ---: | ---: |
| GPABENCH2-CS 训练集 | 0.95 / 0.80 | 0.88 / 0.72 | 0.52 / 0.10 |
| GPABENCH2-CS 测试集 | 0.78 / 0.72 | 0.70 / 0.65 | 0.08 / 0.03 |
| ObfuscationBench-xsum (新闻) | 0.90 / 0.80 | 0.80 / 0.73 | 0.46 / 0.24 |
| ObfuscationBench-squad (专业知识) | 0.90 / 0.82 | 0.81 / 0.76 | 0.37 / 0.17 |
| ObfuscationBench-writing (自由写作) | 0.97 / 0.95 | 0.92 / 0.87 | 0.63 / 0.55 |
