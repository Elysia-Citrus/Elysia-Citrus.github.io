---
title: 第六集：Instruct GPT
published: 2026-09-22
description: '论文精读的开端'
image: ''
tags: [人工智能]
category: '吹水'
draft: false 
lang: ''
---


[GPT-3](https://arxiv.org/abs/2005.14165)已经展示了in-context learning，但是在数学、符号推理、常识推理等多步骤上有一个明显的缺陷，模型必须从题目直接跳到答案，比如你问AI我有10个哈基米，每个哈基米每天哈气0.5小时，一周总共哈多少小时？这种普通的prompt可能直接生成50小时这种错误的答案。

2022年5月24日，Aran在推特上发了个帖子，分享了文章《[Large Language Model are Zero-shot Reasoners](https://arxiv.org/abs/2205.11916)》，说是只要你提出问题之后加一句“Let's Think step by step”，你就立即可以在两个比较难的数学数据集上涨点，而且涨点很显著。由于这句话非常简单，而且效果显著，立刻引起了社区的关注。他们的工作非常朴素，就是在答案后面接一句“让我们一步步来想”，然后在 MultiArith数据集上涨点从17.7%涨到7837%、在GSM8K数据集上从10.4%涨到40.7%. 在之后的工作中，Jason Wei等人的工作[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)改变了few-shot 的形式，他们对QA的答案进行了链式拆解，任务从单纯的 $x→y$ 变成了 $x→z_1 → →z_2→...→y$ ，并且发现COT的收益主要出现在足够大的模型上，较小的模型不会因为思维链边长而变聪明。

不过CoT在最初还有个槽点，那就是人工去写思维链太烦了，而且示例质量会影响结果。因此Zhuosheng Zhang、Aston Zhang、Mu Li（我推的李沐）、Alex Smola在2022年10月提出了[Auto-CoT](https://arxiv.org/abs/2210.03493)，简而言之是让模型自己给一批题目生成reasoning，然后从中挑选有代表性的示例，最后拿这些自动生成的Q+reasoning给新的问题做few-shot prompting，而且它在十个reasoning benchmark 上能达到甚至超过人工设计CoT的表现。

归根结底，如何去解释这个现象呢？人们普遍认为是这些中间的思考促使模型花费更多的算力在中间步骤正确性上，就好比人拿草稿纸演算总比心算要准确一些。而[Transformer](https://arxiv.org/abs/1706.03762)本身也是应用于时序的架构，这不禁令人联想到GRU为什么没能替代LSTM，是因为后者显式更新的cell state所携带的状态信息会更多。而Trsnformer本身能用注意力机制不断更新压缩状态，使得后来生成的token也可以变成反复读取的工作区并成为新的外部状态。复杂序列计算的能力，很大程度上取决于模型能否保存、访问和更新中间状态。那么从这个角度来看，今天所谓的memory、harness，本质上还是在回答这个古老的问题，那就是一个资源有限的计算系统，怎样保存足够多的中间状态，使得后续计算能够继续？这个状态从向量一步步演化为了上下文，乃至外部的存储。

假设 2022 年的你已经理解 GPT-3，并且拿得到 `text-davinci-002` 这类模型，你完全可能在用AI写作业的时候让它一步步把过程写出来，然后意外发现效果比直接给答案要好。实际上在 2021 年，[Scratchpad](https://arxiv.org/abs/2112.00114) 工作已经证明类似的直觉是有效的，只是后来的工作者证明这句话并不只是针对某一道题或者某个领域有效、或是纯粹的偶然，而是在逻辑、算术、符号推理上都奏效，这中间还是差了点系统的科研工作。这样看来，如果我回到2022年，没准也可以发NeurIPS了。

回顾CoT，它留下的主要是四个思想。第一是Inference-time computation，模型在参数不变的情况下，可以通过分配更多的计算资源取得更好的成绩；第二是reasoning trajectory（推理轨迹），也就是不只最终答案值得被优化，中间的思考路径也可以被量化地衡量和改进，启发了后来的Self-Consistency、Tree of Thoughts；第三是Thought+Action，这几乎已经是今天Agent loop的祖先了；第四是不再提示模型推理，而是直接训练模型学会推理。

普通的CoT只是让推理轨迹直接抵达结果 $\tau → y_1$，在后来的工作中，[Self-Consistency](https://arxiv.org/abs/2203.11171)采样了很多条推理轨迹 $\tau_1, \tau_2, ..., \tau_N$  ，并得到多个结果 $y_1, y_2, ..., y_n$ ，最后选择最一致的答案 $y = \text{arg} \max_y \sum_i  \text{1}[\text{Answer}(\tau_i)=y]$  。原论文在GSM8K上相较普通CoT提升了17.9个百分点，在多个算术和常识推理基线上都有明显的改善。当然现在来看这个想法很简单了，就是同一道题多想几次然后投票嘛，那看来我也能和NeurIPS同起同坐了。但是它的思想意义在于，sampling以前常常被人为是语言生成中的随机性，但是现在看来随机性本身可以变成计算资源，不一定让单词推理足够的完美，也可以让模型利用统计规律在很多个不完美的推理里找答案。这对今天的许多工作还是有启发的。对于社区而言这当然是很有趣的，因为不用重新训练模型，几乎是免费的情况下以很少的代价做到了算法增益。

与Self-Consistency同一个月的还有一篇论文[STaR：《Bootstrapping Reasoning With Reasoning》](https://arxiv.org/abs/2203.14465)，它的意思是说让模型先生成reasoning，把答对的reasoning收集起来，然后拿模型自己生成的成功推理再训练模型，如果答错还可以把正确答案告诉它，让它尝试生成一个能导向正确答案的基本原理，不断循环。它的意思是说把搜索好的reasoning写回权重，虽然说它在当时没有Self-Consistency那么简单粗暴地小力出奇迹，但是思想还是蛮重要的，因为搜索更多的reasoning和把搜索好的reasoning写回权重在后续的推理模型时代是一个很重要的思想。

不过Self-Consistency还是一个串行的设计，依旧是一路生成到底，加入在第三步发生了错误，那也得等到生成完整链路之后才知道了。与Self-Consistency不同的是，[Tree of Thought](https://arxiv.org/abs/2305.10601)做了一件很符合传统AI的时期，它把模型产生的n个可能的thought进行评分，保留有希望的节点，然后再继续向下展开，然后产生branch、lookahead、evaluation、backtracking...这是经典搜索算法的牢玩家了。在Game of 24, GPT-4的CoT成功率只有4%，但是ToT达到了74%，这说明LLM不一定非得把思考组织成句子流，也可以把LLM当成生成器，然后用经典算法负责搜索。神经网络刚出来那会大家感觉全完了，MLP大人横扫一切了，结果到了2023这些经典算法又回潮了，比如ToT之后的工作[RAP](https://arxiv.org/abs/2305.14992)明确把Reasoning描述为Planning，并且使用了蒙特卡洛树搜索，让LLM同时扮演世界模型和推理agent。然而社区后来发现搜索不是免费的，令分支因子为 $b$ , 深度为 $d$ ，那么树搜索的复杂度 $O(b^d)$ 会爆炸增长。而且谁来评判哪个thought好？如果让同一个LLM自己给自己达芬，那么生成器与判别器的错误可能高度相关，所以后续大家的研究思路也就转向导Verifier和Reward Model了。

CoT的另一个缺陷是，参数里的记忆决定了后续的推理。如果模型所认定的某个事实是错误的，例如模型以为成龙参演过《大马蜂》，那么后续的推理和答案便都是错误的。2022年10月，Shunyu Yao等人在[ReAct](https://arxiv.org/abs/2210.03629)力提出 $\text{Reasoning + Acting}$ ，此时模型第一步是Thought: "我须要先找到这部电影的信息"，然后调用工具搜索《大马蜂》的演员，获取结果，然后给出反馈。这里边的几个动作：Thought, Action, Observation, Action, Observation... 其中最重要的是搜索，ReAct原论文在HotPotQA、FEVER里调用Wikipedia API来减少CoT的幻觉和误差传播，同时在ALFWorld 和 WebShop等交互任务中让模型进行环境行动，这几乎已经是今天Agent loop的模样了。这种reasoning必须能够被环境反馈修正的架构原则使得让模型思考更久的高度再次提高了一层。有趣的是ReAct和ToT的第一作者都是Shunyu Yao，这哥们从两个方向同时攻击CoT，ReAct让reasoning向外接触世界，ToT让reasoning从内部向内搜索。

再到后来，研究发现模型写出的解释可能收到隐藏bias的影响，但是不在CoT里承认这些因素。换言之，模型可能会受到某个偏差影响得出答案，然后生成一套听起来非常合理的事后诸葛亮。而且让模型自己review自己也没想象中那么可靠，分数反而可能会变低。

在2024年，ChatGPT o1出现了。在此之前，人们并没有关注模型本身，而是在模型之外搭建脚手架。而o1的思路很激进：为什么不予直接训练模型学会这些推理？[OpenAI的技术报告](https://openai.com/index/learning-to-reason-with-llms/)表明，o1通过大规模强化学习会更有效地使用chain of thought，包括发现错误后修正、拆分问题，以及在当前路线失败时换一种策略；性能同时随着更多 RL train-time compute 和更多 test-time thinking compute 提升。但是byd CloseAI并没有披露再多的细节，而2025年[DeepSeek-R1/R1-Zero](https://arxiv.org/abs/2501.12948)开源对社区简直是一次原子弹爆炸的震撼。R1-Zero 直接在基础模型上进行大规模强化学习，并且没有事前做推理监督微调，就出现了许多长程度的推理行为。尽管出现了可读性差、语言混杂等问题，R1在后续又加入了冷启动数据与多阶段训练进行了改善。这让更多研究者相信，长程度的推理行为是可以通过奖励优化学习出来的。于是社区的关注从大家可能听到的prompt工程转向了怎么设计强化学习、激励机制等方向。再到后来，2025年[OpenAI o3、o4mini](https://openai.com/index/introducing-o3-and-o4-mini/)已经能在强化学习中学习什么时候以及怎么学习工具，比如自行搜索网页、调用python、处理图片、改变路线，也就是说ReAct从提示词工程变成了学习行为。

从今天来看，CoT表明token不止是输出，也可以是计算状态，如何管理和利用计算状态对模型的输出质量有直接的影响；Self-Consistency和ToT表明推理不必是一条轨迹，也可以是一个计算的过程，可以用采样、搜索、比较、回退去管理；ReAct令社区发现计算过程不必封闭，也可以与世界交轨；Reasoning Model（后续会介绍）则让人们意识到，"如何进行计算"本身也可以学习。如今各种agent、Harness会决策哪些任务并行、何时调用工具、子智能体如何通信，实际上也是多个子系统管理的过程，llm不再是以前暴力出奇迹的史前时代了。