---
marp: true
size: 16:9
theme: am_green
paginate: true
headingDivider: [2,3]
footer: \ *组会汇报* 
header :   ![#l](../images/logo0.png)
math: mathjax
---

<!-- _class: cover_c -->
<!-- _header: ![#l h:100](../images/logo.png)-->
<!-- _paginate: "" -->
<!-- _footer:   -->

# <!-- fit -->学期总结：LUMIN Proactive TCP Recovery for Seamless WiFi Roaming

###### Enabling Low-Latency Mobile Applications in Indoor Edge Environments
Reporter ：Zhang zheyuan 
Date ：2026-01-18

## 目录

<!-- _class: cols2_ol_ci fglass toc_c  -->
<!-- _footer: "" -->
<!-- _header: "CONTENTS" -->
<!-- _paginate: "" -->

- [Introduction](#3)
- [Evaluation](#6)
- [Design and Fix](#10)
- [Conclusion](#13)


## 学期总结
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **Summary** *introduction* *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

- 完成论文阅读与汇报
- 理解 Lumin 设计思路与实现细节
- 复现 Lumin 仿真结果
- 发现 Lumin 设计缺陷并提出改进方案
- 实现并验证改进方案效果

## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->


#### Real-World Impact

- **Visual SLAM**: Accuracy drops from 
    **90% → 15%** if latency > 666 ms
- **Robots move at 0.5 m/s** 
  → Handoff every **24–50 sec**
- **MAC-layer handoff**: ~100 ms (acceptable)
- **TCP recovery time**: **896 ms** (unacceptable)

> “Seamless roaming preserves *connectivity*, not *performance*.”



## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![image-20260106133710869](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106133710869.png)

## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![image-20260106133746539](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106133746539.png)


## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![image-20260106133810891](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106133810891.png)


## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![image-20260106133929769](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106133929769.png)


## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

#### 三个组件

1. 预测 （通过 RSSI 实现）
2. 降速  （和 1 配合使用）
3. Recovery （快速恢复 TCP 连接）


## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->


#### 修复测量逻辑

|       维度       |  **1. 初始测量逻辑 (Old)**   | **2. 修复后的测量逻辑 (Current)** |                         **改进意义**                         |
| :--------------: | :--------------------------: | :-------------------------------: | :----------------------------------------------------------: |
|   **核心指标**   | **平均延迟 (Average Delay)** |   **完成时间 (Flow Duration)**    | **更直观**：反映用户感知的“传完文件要多久”，而非抽象的单包延迟。 |
|   **发送模式**   |   固定时间 (200s) 无限发包   |      **固定数据量 (12 MB)**       | **消除偏差**：Handoff 会导致发送暂停。固定时间模式下，Mobile 场景发的包少，反而可能拉低平均延迟；固定数据量强制所有场景完成相同任务，Handoff 代价直接体现为时间延长。 |
## 1. Introduction: The Hidden Latency in “Seamless” Roaming

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **introduction** *Evaluation* *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->


#### 修复测量逻辑
|       维度       |  **1. 初始测量逻辑 (Old)**   | **2. 修复后的测量逻辑 (Current)** |                         **改进意义**                         |
| :--------------: | :--------------------------: | :-------------------------------: | :----------------------------------------------------------: |
| **Handoff 控制** |    随机发生，可能在传输外    |  **强制命中 (Velocity=3.0m/s)**   | **确保有效性**：通过调整速度，将 Handoff 时间点控制在 17.8s 左右，确保其发生在 30s+ 的传输窗口中心，确切捕捉切换开销。 |
|   **结果计算**   | `(Mobile延迟 - Static延迟)`  |    `(Mobile时间 - Static时间)`    |    **量化开销**：直接计算 Handoff 导致传输多花了多少秒。     |



## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

#### 结果分析1


**实验配置**：

- **总数据量**：12 MB (12,582,912 Bytes)
- **应用类型**：TCP Upload (BulkSendApplication)
- **移动速度**：3.0 m/s (快速移动场景，确保 Handoff 快速发生)
- **Handoff 时间点**：约 17.78s


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

| 指标                   | **Scenario 1: Ideal**(无 Handoff) | **Scenario 2: Baseline**(普通策略) | **Scenario 3: Lumin**(优化策略) |
| ---------------------- | :-------------------------------: | :--------------------------------: | :-----------------------------: |
| **传输完成时间 (s)**   |            **32.00 s**            |            **37.21 s**             |           **40.62 s**           |
| **Handoff 开销 (s)**   |                 -                 |            **+5.20 s**             |           **+8.61 s**           |
| **丢包数量 (Packets)** |                 0                 |                101                 |               143               |
| **Handoff 时刻**       |                N/A                |              17.78 s               |             17.78 s             |
| **平均吞吐量**         |             3.15 Mbps             |             2.70 Mbps              |            2.48 Mbps            |
## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

#### 结果分析：为什么 Lumin 耗时更长？

**主动降速 (Rate Control) 的代价**：

- **Lumin**：预测到 RSSI 衰减后，为了防止拥塞，主动大幅降低了发送速率（cwnd）。对于大文件传输，**降速 = 放弃带宽 = 延长时间**。
- **Baseline**：虽然 Handoff 时卡顿了 5.2s（断连+重连），但在断连前的每一秒它都在全速发送，抢出了更多数据。

**激进恢复的副作用**：

- **Lumin**：在重连瞬间（17.78s）立即强制重传。数据通过丢包数（143 vs 101）表明，这种激进策略在链路刚恢复时不稳，导致了**二次丢包**，反而拖慢了 TCP 的恢复。


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![image-20260106120055622](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106120055622.png)


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->


![#c h:500](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106120832940.png)


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

**为什么满足条件（黄色/绿色区域开始）是在 13.8s，但直到 17.78s 才真正切过去？**

这就解释了为什么 Baseline 的效果不好，因为存在巨大的 **滞后 (Lag)**：

1. **扫描延迟 (Scanning Delay)**： 即使逻辑上判断要切换，STA（终端）必须先启动扫描（Active Scanning）来“发现” AP2。这个过程在仿真器中需要几百毫秒甚至更久。
2. **信标丢失机制 (MaxMissedBeacons)**： NS-3 的 `StaWifiMac` 默认行为非常保守。它往往不会仅仅因为“信号弱”就主动断开，而是会死守 AP1，直到连续丢失了 **10个 Beacon 包**（约 1秒）确认“彻底没救了”才会断开。
3. **证据**：日志显示断连发生在 RSSI 跌到 **-82dBm** 之后，这是典型的“死守到底”行为。
4. **决策间隙**： 从 13.8s (满足阈值) 到 17.8s (断连)，中间这 **4秒** 就是用户体验最差的“弱信号卡顿期”。Lumin 策略正是在这期间起作用——它预测到了这个趋势，虽然它没能提前强制切换（受限于 MAC 层实现），但它**提前降速**了，防止了这 4秒 内发太多的包导致大规模丢包。
- **结论**： 当前仿真环境复现了真实的 WiFi **“粘滞效应” (Sticky Client)** —— 即终端死守弱信号 AP 不放。


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

![#c h:500](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106122005022.png)

## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

| 指标                   | **Scenario 1: Ideal**(无 Handoff) | **Scenario 2: Baseline**(普通策略) | **Scenario 3: Lumin**(优化策略) |
| ---------------------- | :-------------------------------: | :--------------------------------: | :-----------------------------: |
| **传输完成时间 (s)**   |            **32.00 s**            |            **32.64 s**             |           **35.29 s**           |
| **Handoff 开销 (s)**   |                 -                 |            **+0.64 s**             |           **+3.29 s**           |
| **丢包数量 (Packets)** |                 0                 |                 1                  |               61                |
| **Handoff 时刻**       |                N/A                |              17.78 s               |             17.78 s             |
| **平均吞吐量**         |             3.15 Mbps             |             2.70 Mbps              |            2.48 Mbps            |

## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

- **Baseline** 在这 4秒 弱信号期继续全速发包，导致最后断连时丢了 101 个包，但抢出了一些流量。
- **Lumin** 在这 4秒 弱信号期侦测到了风险，**主动把速度降到了极低**。这导致它没能在断连前抢发数据，从而延长了总时间，但也验证了其“预测与控制”逻辑是生效的。





#### 什么时候触发降速

**只要有数据包接收，Lumin 就会实时预测 RSSI**。 它会计算一个 **切换概率 (`p_predict`)**，这个概率由两个因素决定：

- **当前信号多差**：距离阈值 (`-80dBm`) 越近，概率越高。
- **目标信号多强**：目标 AP 比当前 AP 强出越多（接近迟滞值 `7dB`），概率越高。

## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->

#### 怎么降速

![#c](https://zzhaire-markdown.oss-cn-shanghai.aliyuncs.com/imgs/image-20260106141315097.png)


## 2. Simulation Evaluation NS-3

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* **Evaluation** *Design and Fix* *Conclusion* -->
<!-- _class: navbar -->
#### 降多少速？

Lumin 试图根据概率动态调整 TCP 的拥塞窗口（Cwnd），公式逻辑如下：
- **平滑调整**：`p = (1.0 - 新概率) / (1.0 - 旧概率)`。如果概率从 10% 涨到 20%，窗口就缩小一点。
- **兜底限制**：代码设置了最少保留 40% 的窗口 (`1.0 - 0.6`)，防止降得太低。
```
if (p_predict > 0.5)
    p = 0;
```
- **激进的“断崖式”降速**：一旦预测的切换概率超过 **50%**（这在 Handoff 前几秒很容易达到），Lumin 会直接把调整因子 `p` 设为 **0**。
- **后果**：TCP 层收到 `p=0` 后，会将发送窗口 (`Cwnd`) 强制重置为 **1 个 MSS**（最小单位）。
  - 这意味着在 Handoff 发生前的几秒钟（比如 RSSI 还是 -78dBm，还没断连时），**Lumin 就已经自我扼杀，几乎停止发送数据了**。







## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->

##### 结果分析3



**场景 A: 距离 40m (困难模式，信号重叠严重)**

|   策略 (Policy)   | 总耗时 (Time) | 丢包 (Lost) |                           评价                           |
| :---------------: | :-----------: | :---------: | :------------------------------------------------------: |
|   **Baseline**    |  **37.21 s**  |     101     |      只要不断连就死命发，虽然丢包多，但抢出了时间。      |
| **Recovery Only** |  **37.02 s**  |     69      |       丢包减少，时间持平。说明快速恢复有一点帮助。       |
|  **Pred. Only**   |  **42.69 s**  |     48      | **严重降速 (+5.4s)**。预测模块为了省流量，把速度降没了。 |
| **Lumin (Full)**  |  **40.62 s**  |     143     |               综合了预测的慢和恢复的激进。               |


## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->

##### 结果分析3

**场景 B: 距离 60m (简单模式，目标信号强)**

| 策略 (Policy)     | 总耗时 (Time) | 丢包 (Lost) |                       评价                        |
| ----------------- | :-----------: | :---------: | :-----------------------------------------------: |
| **Baseline**      |  **32.64 s**  |    **1**    |   **表现完美**。Handoff 一瞬间完成，几乎无损。    |
| **Recovery Only** |    34.56 s    |      1      |    稍慢一点，可能是重传机制在好网络下的误判。     |
| **Pred. Only**    |  **56.23 s**  |     71      | **灾难级表现 (+23.6s)**。完全被“预测逻辑”坑死了。 |
| **Lumin (Full)**  |    35.29 s    |     61      |  被 Recovery 救回来一点，但依然受预测模块拖累。   |


## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->

#### 场景：网络较好

在 60m 距离设置下，Handoff 发生时，目标 AP 的信号非常强（-66dBm）。这意味着：

- **切换极快**：几乎没有断连时间。
- **没丢包**：数据包只是在切换那几毫秒里**迟到**了，并没有真正丢失。


## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->

#### Recovery 错误操作

Lumin 的 `Seamless Recovery` 机制是一个“无脑”触发器：

- **逻辑**：只要连上新 AP (OnAssociated)，不管有没有丢包，立刻强制 TCP 重传所有未确认的数据 (

  ForceRetransmit)。

  

  在 60m 场景下，Recovery 策略反而变慢（34.5s vs Baseline 32.6s），是因为它犯了一个**“用力过猛”**的错误。

  简单来说：**它把“没丢的包”当成“丢包”重传了，导致 TCP 误以为网络拥塞降速。**

- **后果**：

  - TCP 发送了一堆**重复数据包 (Duplicate Packets)**。
  - 因为原来的包其实已经到了（或者马上就到了），接收端收到了两份一样的数据。

## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->


#### 触发拥塞控制

接收端收到重复包后，会回传 **重复确认 (Duplicate ACKs / D-SACKs)**。 发送端收到这些 DupACKs 后，会做出误判：

1. **误判**：“哎呀，收到重复确认，说明网络有丢包/拥塞了！”
2. **惩罚**：触发 **快速恢复 (Fast Recovery)** 算法。
3. **降速**：将拥塞窗口 (**Cwnd**) **减半** (Halving Cwnd)。



## 3. Design and Fix

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* **Design and Fix** *Conclusion* -->
<!-- _class: navbar -->
#### 对比 Baseline

- **Baseline**：
  - 切换后，发现没有丢包（或者 ACK 正常回来），TCP **保持现有的大窗口** 继续全速发送。
  - 结果：速度不减，跑得飞快。
- **Recovery**：
  - 主动重传 -> 触发误判 -> **窗口减半** -> 慢慢爬坡恢复。
  - 结果：因为窗口被砍了一刀，反而浪费了 2 秒钟去重新提速。



## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->

#### 修改

**删除或修改这个 `p=0` 的逻辑**，改为更温和的降速（例如最低保留 10% 的窗口，而不是直接归零），让它在切换前能继续传输数据。

优化后的 Lumin 策略成功融合了两个模块的优点，并互补了缺点：

- **Predictive Rate Control (预测降速)**：

  - **作用**：在 Handoff 前主动降低 TCP 窗口。
  - **贡献**：在 40m 场景下，将丢包从 101 个减少到 60 个（**降幅 40%**）。
  - **优化点**：我们移除了 `p=0` 的自杀式逻辑，即使在最危险时刻也保留 10% 的带宽，避免了“Pred. Only”出现的 56s 极端延迟。


## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->

- **Seamless Recovery (无缝恢复)**：

  - **作用**：在 Handoff 后立刻“踢”一脚 TCP。
  - **贡献**：在 60m 场景下，帮助 Lumin 跑出了比 Baseline 还快 1.2s 的成绩。因为它消除了 TCP 在重连后等待 RTO（通常 1 秒）的死区时间。
  - **优化点**：我们实施了 **“Conservative Recovery” (仅重传首包)**。这避免了在好网络下因为重复发包而导致 TCP 误判拥塞（之前 Recovery Only 变慢的原因）。

  

## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->


#### 消融实验

| 场景              | 策略 (Policy)     | 模块组成        | 总耗时 (Time) | 丢包 (Lost) | 性能评价                                                     |
| :---------------- | :---------------- | :-------------- | :------------ | :---------- | :----------------------------------------------------------- |
| **60m**(优良网络) | **Baseline**      | 无              | 32.64 s       | 1           | **基准**。切换快，无丢包。                                   |
|                   | **Pred. Only**    | 仅预测降速      | 56.22 s       | 14          | **严重变慢**。降速后缺乏快速恢复机制，导致 TCP 长期维持低速。 |
|                   | **Recovery Only** | 仅快速恢复      | 32.71 s       | 1           | **正常**。修复了之前的“虚假重传”副作用，与 Baseline 持平。   |
|                   | **Lumin (Full)**  | **预测 + 恢复** | **31.45 s**   | **1**       | **最优 (-1.2s)**。降速减少了积压，恢复机制消除了 RTO 等待。  |


## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->


#### 消融实验

| 场景              | 策略 (Policy)     | 模块组成        | 总耗时 (Time) | 丢包 (Lost) | 性能评价                                                     |
| :---------------- | :---------------- | :-------------- | :------------ | :---------- | :----------------------------------------------------------- |
| **40m**(恶劣网络) | **Baseline**      | 无              | 37.21 s       | 101         | **严重丢包**。死磕到底的策略导致大量重传。                   |
|                   | **Pred. Only**    | 仅预测降速      | 42.75 s       | 26          | **丢包最少**。降速非常有效，但因缺乏恢复机制，整体传输慢。   |
|                   | **Recovery Only** | 仅快速恢复      | 37.60 s       | 122         | **无效**。在高丢包场景下，仅靠恢复无法解决拥塞根本问题。     |
|                   | **Lumin (Full)**  | **预测 + 恢复** | **36.59 s**   | **60**      | **综合最优**。比 Baseline 快，且丢包减少了 **40%**。         |

## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->

#### 为什么 Pred. Only 单独跑很慢？

- 它虽然成功降低了速率（少丢包），但它把 TCP 窗口缩得太小了。
- 切换完成后，如果没有 Recovery 模块去“踢”它，TCP 需要很长时间（慢启动）才能把窗口重新涨回来。
- **结论**：降速（防守）必须配合 快速恢复（进攻）一起使用，缺一不可。

## 4. Fix and conclusion

<!-- _header: \ ***![#l h:40](../images/logo.png)*** *introduction* *Evaluation* *Design and Fix* **Conclusion** -->
<!-- _class: navbar -->


1. **在好网络下 (60m)**：它利用“快速恢复”消除 RTO 等待，速度超越 Baseline。
2. **在差网络下 (40m)**：它利用“预测降速”大幅减少丢包，同时保持了传输效率。





---
<!-- _class: lastpage  -->
<!-- _header: ![#l h:40](../images/logo.png)-->

###### Q & A !

<div class = "icons">

</div>
