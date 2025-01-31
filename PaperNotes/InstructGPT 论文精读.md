# InstructGPT

**Training language models to follow instructions with human feedback**
Long Ouyang et al. | [PDF](https://arxiv.org/pdf/2203.02155) | [精简版](https://openai.com/index/instruction-following/) | OpenAI | 2022.03

> **学习 & 参考资料**
>
> - **相关文章**
>
>   - [Transformer 论文精读](./Transformer%20论文精读.md)
>   - [GPT 论文精读](./GPT%20论文精读.md)
>
> - **机器学习**
>
>   —— 李宏毅老师的 B 站搬运视频
>
>   - [ChatGPT (可能)是怎么炼成的](https://www.bilibili.com/video/BV1TD4y137mP/?p=6&share_source=copy_web&vd_source=e46571d631061853c8f9eead71bdb390)
>
> - **论文逐段精读**
>
>   —— 沐神的论文精读合集
>
>   - [InstructGPT 论文精读【论文精读·48】](https://www.bilibili.com/video/BV1hd4y187CR/?share_source=copy_web&vd_source=e46571d631061853c8f9eead71bdb390)
>
> 这是一篇重要的文章，在 [ChatGPT](https://openai.com/index/chatgpt/) 的 Method 中有提及：
>
> “We trained this model using Reinforcement Learning from Human Feedback (RLHF), using the same methods as [InstructGPT⁠](https://openai.com/index/instruction-following/), but with slight differences in the data collection setup. ”
>



## 数据集

### 数据来源

- **标注人员编写的提示**

  在初始阶段，为了训练第一个 InstructGPT 模型，研究团队专门聘请了标注人员来编写指令式（instruction-like）提示，因为这种类型的提示在常规 GPT-3 模型的 API 数据中并不常见。

  要求标注人员编写以下三种类型的提示：

  - **普通（Plain）**：编写任意任务，需要确保任务多样性

  - **少样本（Few-shot）**：提供一条指令以及多个问/答（query/response）示例对。

  - **基于用户（User-based）**：根据 OpenAI API 等候列表用户的应用需求，设计相关任务提示。

    > [OpenAI ChatGPT API Waitlist](https://share.hsforms.com/1u4goaXwDRKC9-x9IvKno0A4sk30) 中有一个必填的栏目 “Are there specific ideas you're excited to build with the ChatGPT API?” ，即“你有什么特别想用 ChatGPT API 实现的想法吗？”

  > [!tip]
  >
  > 论文第 7 页的 3.4「Human data collection」 和第 36 页附录 B「Additional human data collection details」详细的阐述了标注人员招聘，指导、任务分配以及数据收集流程等，如果有实际需要可以参考。

- **API 收集的提示**

  > *“Our prompt dataset consists primarily of text prompts submitted to the OpenAI API, specifically those using an earlier version of the InstructGPT models (trained via supervised learning on a subset of our demonstration data) on the Playground interface.”*

  早期的 InstructGPT 模型被部署到 [Playground](https://beta.openai.com/playground) 中（新用户注册后会赠送一些 API 额度），当时用户和 InstructGPT 模型交互时会收到通知，告知其数据可能被用于模型的进一步训练（不包含来自生产环境的 API 用户数据）。

  收集到的数据做以下处理：

  1. **去重**：通过启发式方法检测并删除长前缀相同的提示。
  2. **限制提示数量**：每个用户的提示最多 200 条。
  3. **数据集划分**：根据用户 ID 划分为训练、验证和测试集，确保「验证/测试集」和「训练集」不存在相同用户（减少可能的数据泄露，因为同一用户可能会重复的询问相同主题的问题）。
  4. **过滤敏感信息**：如检测到个人身份信息（personally identifiable information，PII）会去除。

### 数据构建

研究团队基于上述提示数据构建了三类数据集，用于不同训练阶段：

| 数据集类型 | 数据规模 | 数据来源                        | 用途             |
| ---------- | -------- | ------------------------------- | ---------------- |
| **SFT**    | 12,725   | 标注人员 (11,295) + API (1,430) | 有监督微调       |
| **RM**     | 33,207   | 标注人员 (6,623) + API (26,584) | 奖励模型训练     |
| **PPO**    | 31,144   | 完全来自 API                    | 强化学习策略优化 |

### 数据分布和示例

> **表 1**
>
> ![表 1](./assets/image-20250118122308319.png)

上表展示了 API 数据集中提示的使用类别分布，其中生成任务占比最高，为 45.6%。论文在附录 A.2.1 提供了一些提示示例，摘选部分进行理解：

| 用例类型                        | 样例                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| Generation<br />**生成**        | Write a short story where a brown bear to the beach, makes friends with a seal, and then return home.<br />写一篇关于一只棕熊去海滩与海豹交朋友然后回家的短篇故事。 |
| Open QA<br />**开放式问答**     | Who built the statue of liberty?<br />自由女神像是由谁建造的？ |
| Brainstorming<br />**头脑风暴** | What are 10 science fiction books I should read next?<br />我接下来应该阅读的 10 本科幻小说有哪些？ |
| Chat<br />**聊天**              | The following is a conversation with an AI assistant. The assistant is helpful, creative, clever, and very friendly.<br /><br />Human: Hello, who are you?<br/> AI: I am an AI created by OpenAI. How can I help you today? Human: I’d like to cancel my subscription.<br/> AI:<br />一段对话 |
| Rewrite<br />**重写**           | Translate this sentence to Spanish:<br /><br /><English sentence><br />将以下句子翻译成西班牙语：<br /><br /><英文句子> |
| Summarization<br />**摘要**     | Summarize this for a second-grade student:<br /><br />{text}<br />为二年级学生总结以下内容：<br /><br />{文本} |
| Classification<br />**分类**    | {java code}<br /><br />What language is the code above written in?<br />上面的代码是用什么语言编写的？ |
| Other<br />**其他**             | start with where<br />从哪里开始                             |
| Closed QA<br />**封闭式问答**   | Answer the following question:<br />What shape is the earth?<br /><br />A) A circle <br />B) A sphere <br />C) An ellipse <br />D) A plane<br />一个选择题。 |
| Extract<br />**抽取**           | Extract all place names from the article below:<br /><br />{news article}<br />从下面的文章中提取所有的地名：<br /><br />{新闻文章} |

## 模型

> InstructGPT 从 GPT-3 的预训练模型（1.3B、6B、175B）开始训练，但预训练模型并不能直接产生人类所期望的回答[^1]：
>
> ![文字接龙（预训练）](./assets/%E6%96%87%E5%AD%97%E6%8E%A5%E9%BE%99.svg)

### 训练概览

> ![图 2](./assets/Methods_Diagram_light_mode.jpg)
>

先通过图示来了解模型的训练过程，分三步（从左到右）：

1. **收集示范数据并进行有监督微调（SFT）**

   - 从提示数据集（prompt dataset）中抽取提示（如“向6岁儿童解释月球登陆”）。

   - 标注人员根据提示撰写期望的答案（如“有些人去过月球……”）

     > ![撰写期望的回答](./assets/%E6%92%B0%E5%86%99%E6%9C%9F%E6%9C%9B%E7%9A%84%E5%9B%9E%E7%AD%94.svg)

   - 这些示范数据被用来有监督微调 GPT-3 模型。

2. **收集对比数据并训练奖励模型（RM）**

   - 针对同一提示，生成多个模型输出（如图中 A、B、C、D 的例子）。

   - 标注人员根据输出质量对这些结果进行排序，从最优到最差（如 D > C > A = B）。

   - 这些对比数据被用来训练一个奖励模型，让它能够正确给输出打分。

     > 询问 ChatGPT 的时候，期望的回答通常是答案，而不是续写这个问题：
     >
     > ![对比输出以及训练奖励模型](./assets/%E5%AF%B9%E6%AF%94%E8%BE%93%E5%87%BA%E4%BB%A5%E5%8F%8A%E8%AE%AD%E7%BB%83%E5%A5%96%E5%8A%B1%E6%A8%A1%E5%9E%8B.svg)
     >
     > 所以在偏好（输出质量）上，「玉山」> 「谁来告诉我呀」。

3. ###### **强化学习（RL）微调**

   - 从提示数据集中抽取新的提示（如“写一个关于青蛙的故事”）。

   - 模型生成输出。

   - 奖励模型对生成的输出进行评分（作为奖励信号）。

   - 采用近端策略优化（PPO）算法对模型进行梯度更新，使其更好地符合人类偏好。

     > 下图中的增强式学习（Reinforcement learning）就是强化学习。
     >
     > - **最初（GPT）**
     >
     >   ![低分](./assets/%E4%BD%8E%E5%88%86.svg)
     >
     > - **强化学习后（ChatGPT）**
     >
     >   ![高分](./assets/%E9%AB%98%E5%88%86.svg)


[^1]: [来自于李宏毅老师「ChatGPT 是怎么炼成的」课件](https://docs.google.com/presentation/d/1vDT11ec_nY6P0o--NHq9col5XEE4tHBw/edit#slide=id.p14)
