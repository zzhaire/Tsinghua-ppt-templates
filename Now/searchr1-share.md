---
marp: true
size: 16:9
theme: am_blue
paginate: true
headingDivider: [2,3]
footer: \ *Paper Sharing* 
header :   ![#l](../images/logo0.png)
math: mathjax
---

<!-- _class: cover_c -->
<!-- _header: ![#l h:100](../images/logo.png)-->
<!-- _paginate: "" -->
<!-- _footer:   -->
# Search-R1: Training LLMs to Reason and Leverage Search Engines with Reinforcement Learning

Published as a conference paper at COLM 2025
Bowen Jin, Hansi Zeng, Zhenrui Yue, Jinsung Yoon, Sercan Ö. Arık, Dong Wang, Hamed Zamani, Jiawei Han

Reporter：Zhang zheyuan 
Date：2026-05-26

## 目录

<!-- _class: cols2_ol_ci fglass toc_c  -->
<!-- _footer: "" -->
<!-- _header: "CONTENTS" -->
<!-- _paginate: "" -->

- [Introduction](#3)
- [Background](#8)
- [Design](#16)
- [Evaluation](#23)
- [Conclusion](#29)


## 1 Introduction & Search-R1 核心摘要 <!-- page 3 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Introduction** *Background*  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

🎯 **问题**：大模型解决复杂问题时（如多跳问答、事实核查），必须动态获取外部知识

⚠️ **核心挑战**：
- 模型参数知识有截止/幻觉风险
- 单次检索无法支撑迭代推理
- "怎么查"比"查什么"更难教

💡 **关键洞察**：让LLM像人一样**自主决策检索时机与内容**，是通向可靠推理的关键一步

> 本次介绍：Search-R1 用强化学习教会模型"边想边查"的后训练的开山之作

## 1.Introduction & 现有方案及其局限 <!-- page 4 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Introduction** *Background* *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->
| 方案 | 核心思路 | 关键缺陷 |
|------|----------|----------|
| **RAG** | 先检索→再生成 | 单次检索，无法迭代修正；检索与推理解耦 |
| **Prompt Tool-use** | 教模型调用搜索 | 泛化弱，依赖高质量标注轨迹 |
| **SFT端到端** | 监督微调交互策略 | 搜索操作不可微，难规模化 |

<br>

> 🔑 核心矛盾：**"怎么查"比"查什么"更难教**，现有方法无法让模型自主学会检索策略

## 1.Introduction & Search-R1 的核心突破 <!-- page 5 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Introduction** *Background*  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

✅ **范式创新**：将搜索引擎纳入RL环境，实现"思考-检索"多轮交替  
✅ **极简设计**：  
  - 仅用结果奖励（Exact Match），无需过程/格式奖励  
  - 检索Token Loss Masking，防止优化漂移  
  - 原生兼容PPO/GRPO算法  

💡 **本质**：不手工设计"怎么查"，让强化学习自主探索最优交互策略

## 1.Introduction & 效果速览：同设置下显著提升 <!-- page 6 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Introduction** *Background*  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

| 模型 | 平均提升 (vs RAG) | 多跳任务最佳提升 |
|------|------------------|------------------|
| Qwen2.5-7B | **+24%** | Musique +238% |
| Qwen2.5-3B | **+20%** | PopQA +12% |

<br><br>

✅ **公平对比**：同检索器 / 同语料 / 同Top-k=3 / 同预训练基座  
✅ **泛化性强**：分布内（NQ/HotpotQA）+ 分布外（5个多跳数据集）均有效  
✅ **模型无关**：Base/Instruct 最终性能趋同 → RL 可弥补预训练差异

## 1.Introduction & 核心问题与突破 <!-- page 7 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Introduction** *Background*  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

🎯 **核心问题**：如何让LLM自主学会"边思考边搜索"的交互策略？

💡 **Search-R1 的做法**：
- 把搜索变成环境的一部分，用RL端到端优化
- 极简奖励（Exact Match）+ Token Masking = 稳定训练
- 不教"怎么查"，让模型自己探索最优策略

## 2.Background & 强化学习 <!-- page 8 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar cols-2 -->
<div class="ldiv">

🎮 **场景**：4×4 网格，智能体从起点走向终点
- **State (状态)**：智能体当前所在位置
- **Action (动作)**：上/下/左/右/原地不动
- **Reward (奖励)**：每步 -1（取决具体场景），到达终点 +10
- **Episode (回合)**：从起点到终止状态的一次完整交互过程

> 💡 核心目标：在 Episode 中最大化**累计奖励**，而非单步奖励

</div>
<div class="rdiv">

<!-- 图片占位 -->

</div>

## 2.Background & 强化学习 <!-- page 9 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar cols-2-64 -->

<div class="ldiv">

- **Trajectory**: $\tau = (s_0, a_0, r_1, \dots, s_T)$ 一次完整交互序列
- **Return**: $G_t = \sum \gamma^k r_{t+k}$ 轨迹上所有奖励之和
- **State Value**: $V^\pi(s) = \mathbb{E}[G_t \mid s_t=s]$ 从状态$s$出发的期望Return
- **Action Value**: $Q^\pi(s,a) = \mathbb{E}[G_t \mid s_t=s, a_t=a]$ 选动作$a$后的期望Return
- **Policy $\pi(a|s)$**: 策略分布（如上下左右的具体概率）
- **Bellman方程**: $V(s) = \mathbb{E}[r + \gamma V(s')]$ 价值递归关系，Reward由环境期望决定

</div>
<div class="rdiv">

<!-- 图片占位 -->

</div>

## 2.Background & 强化学习——值迭代 <!-- page 10 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

值迭代算法：从随机初始值到最优策略

**算法流程**：
1. **初始化**：任意设定 $V_0(s)$（如全为0）
2. **迭代更新**（直到收敛）：
   - **Q值计算**：$Q_k(s,a) = \mathbb{E}[r|s,a] + \gamma \sum_{s'} P(s'|s,a) V_k(s')$
   - **策略改进**：$\pi_{k+1}(s) = \arg\max_a Q_k(s,a)$ （贪心选择最优动作）
   - **价值更新**：$V_{k+1}(s) = \max_a Q_k(s,a)$ （贝尔曼最优方程）

> 💡 **数学保证**：当迭代次数 $k \to \infty$，$V_k(s)$ 收敛到最优价值 $V^*(s)$，策略收敛到最优策略 $\pi^*$

## 2.Background & 强化学习——蒙特卡洛近似 <!-- page 11 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

- **问题**：$V^\pi(s)=\mathbb{E}[G_t\|s]$ 的期望无法直接计算（状态空间太大）
- **蒙特卡洛思路**：用**多次rollout的样本平均**近似期望
  - 一次 Rollout：从状态$s$出发，按策略$\pi$采样一条完整轨迹$\tau$，计算$G_t$
  - 重复$N$次：$\hat{V}(s) \approx \frac{1}{N}\sum_{i=1}^N G_t^{(i)}$

⚠️ 局限：方差大、必须等到回合结束才能更新
> 数学上可以证明，即使是小的采样，最后算法也可以收敛到最优值
> 证明可以参考《强化学习的数学原理》 赵世钰

## 2.Background & 强化学习——神经网络近似 <!-- page 12 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar bq-red-->

- **瓶颈**：现实问题状态/动作空间巨大（如围棋 $10^{170}$），表格法无法存储与更新
- **核心思想**：用神经网络作**万能函数**，替代查表
- **参数化表示**：
  • 价值网络 $V_\phi(s)$：输入状态 → 输出期望回报
  • 策略网络 $\pi_\theta(a\|s)$：输入状态 → 输出动作概率分布
- **优势**：具备**泛化能力**，能评估未见过的状态，支撑大规模RL训练

> 大模型推理过程也可以看作一个RL过程：
> 
> 💡 大模型的promt + token1, token2, ... token n 可以记作当前的 **State**
> 💡 大模型自回归生成一个token记作 **Action**
> 💡 大模型自回归生成每个token的概率为 **policy**

## 2.Background & 强化学习——Actor-Critic <!-- page 13 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

- **核心思想**：两个神经网络分工，兼顾"决策"与"评估"
  - **Actor (策略网络)**：输入状态 $s$，输出动作概率 $\pi_\theta(a\|s)$，负责"怎么选"
  - **Critic (价值网络)**：输入状态 $s$，输出价值估计 $V_\phi(s)$，负责"评好坏"
  - **TD误差**：$\delta = r + \gamma q_w(s',a') - q_w(s,a)$  
- **优势函数 (Advantage)**：$A_t = r_t + \gamma V(s_{t+1}) - V(s_t)$，衡量动作"比预期好多少"
- **更新机制**：Actor 沿 $A_t$ 梯度上升优化策略，Critic 用 TD 误差拟合价值

> 💡 双网络配合：Actor 探索方向，Critic 降低方差，实现高效在线学习

## 2.Background & 强化学习——Actor-Critic <!-- page 14 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

**算法**：在每个episode的时间步$t$执行

1. **采样**：根据$\pi(a\|s_t,\theta_t)$生成$a_t$，观察$r_{t+1}, s_{t+1}$
2. **预采样**：根据$\pi(a\|s_{t+1},\theta_t)$生成$a_{t+1}$（用于bootstrap）

3. **Critic更新（Value Update）**：
   $$w_{t+1} = w_t + \alpha_w [r_{t+1} + \gamma q(s_{t+1},a_{t+1},w_t) - q(s_t,a_t,w_t)] \nabla_w q(s_t,a_t,w_t)$$

4. **Actor更新（Policy Update）**：
   $$\theta_{t+1} = \theta_t + \alpha_\theta \nabla_\theta \ln \pi(a_t\|s_t,\theta_t) q(s_t,a_t,w_{t+1})$$

> ✅ 双网络交替更新：Critic先学，Actor再学

## 2.Background & 强化学习——Critic 网络如何估计 Action Value <!-- page 15 -->

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *Introduction* **Background**  *Design* *Evaluation* *Conclusion* --> 
<!-- _class: navbar -->

- **问题**：$Q^\pi(s,a)$ 无法直接计算，需用神经网络近似
- **参数化表示**：$q_w(s,a)$ —— 输入 $(s,a)$，输出标量价值估计
- **训练目标**：最小化 TD 误差的平方
  $$L(w) = \mathbb{E}[(r + \gamma q_w(s',a') - q_w(s,a))^2]$$
- **梯度更新**：
  $$w_{t+1} = w_t + \alpha_w [r + \gamma q_w(s',a') - q_w(s,a)] \nabla_w q_w(s,a)$$

> 💡 Critic 通过"预测→观察实际→修正预测"循环，逐步逼近真实 $Q$ 值

<!-- END_PART_1 -->
