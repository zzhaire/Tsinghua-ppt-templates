---
marp: true
size: 16:9
theme: am_purple
paginate: true
headingDivider: [2,3]
footer: \  *互联网体系结构及其安全基础* 
header :   ![#l](../images/logo0.png)
math: mathjax
---

<!-- _class: cover_a -->
<!-- _header: ![#l h:100](../images/logo0.png)-->
<!-- _paginate: "" -->
<!-- _footer: 互联网体系结构及其安全基础 -->

# <!-- fit -->基于 [USENIX Security '23] NRDelegationAttack 的研究复现 DNS 复杂度攻击

###### NRDelegationAttack: Complexity DDoS attack on DNS Recursive Resolvers
Reporter ：田思源 张哲源
Date ：2025 年 12 月 21 日

## 目录

<!-- _class: cols2_ol_ci fglass toc_c  -->
<!-- _footer: "" -->
<!-- _header: "CONTENTS" -->
<!-- _paginate: "" -->

- [论文提要](#3)
- [攻击模型](#6)
- [实验设计](#10)
- [实验分析](#12)
- [讨论与分工](#13)


## 1. 论文提要：从“一个补丁”到“另一个补丁”

<!-- _header: \ ***![#l h:40](../images/logo.png)*** **论文提要** *攻击模型* *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2 -->

<div class = "ldiv">

##### 背景知识

**DNS 递归查询 (Recursive Query):**

- 用户向解析器发起查询，解析器代为完成全部查询过程，直到返回最终 IP 地址。
- 解析器通过逐级询问（根 → TLD → 权威）和引荐响应（NS 记录）找到目标服务器。

**引荐响应 (Referral Response, RR):**
- 当一个权威服务器（如 `.com` 服务器）无法提供最终答案时，它会“引荐”解析器去问下一个服务器（如 `example.com` 的服务器）。

- 这个响应中包含一个 **NS 名称列表**（Name Server list），告诉解析器“你应该去问谁” 。



</div>
<div class = "rdiv">

**胶水记录 (Glue Records):**

- "胶水记录" 是指在引荐响应中**附带**的 NS 名称所对应的 **IP 地址**。

> 本文提出了一种针对 DNS 解析器的 **“复杂度”攻击** (complexity attack) 。

**攻击的关键前提：无胶水记录 (No Glue Records)**


![#c h:200](assets/basic-final/image.png)
<p class="caption" align="center">图 1: DNS 解析器的工作流程</p>



## 1. 论文提要：从“一个补丁”到“另一个补丁”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **论文提要** *攻击模型* *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64 bq-green -->

<div class = "rdiv">

> NXNSAttack 补丁
> 
>   为缓解 NXNSAttack，现代解析器引入了 **“引荐响应限制”**<br>
>   **这个“补丁”规定：** 无论引荐列表（LRR）有多长（例如 $n=1500$），解析器一次**只启动** $k$ 个（例如 $k=5$ 或 $k=6$）NS 名称的解析 。<br>
>   在启动这 $k$ 个解析后，解析器会设置一个 `"No_Fetch"` 标志，表示“已达到本轮限制”。
</div>
<div class ="ldiv">

#### 补丁 1：NXNSAttack (洪水攻击)

- **攻击原理：**
  - 攻击利用上一步的 LRR（大型列表、无胶水记录），并让所有 $n$ 个 NS 名称都指向 **“不存在”的 (NXDOMAIN) 域名**。
- **触发“洪水” (Flood)：**
  - 当一个**未打补丁**的解析器收到这个 LRR 时，它会**同时启动 $n$ 个（例如 1500 个）解析进程**，去查询那些“不存在”的域名。
- **后果：**
  - 这一个恶意查询，就引发了**海量的对外 DNS 查询**(数据包)



## 1. 论文提要：从“一个补丁”到“另一个补丁”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **论文提要** *攻击模型* *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar  -->
#### 补丁 1：NXNSAttack (洪水攻击)

![#c h:400](assets/basic-final/image-1.png)

<p align="center" class="caption">图 2: NXNSAttack 攻击流程</p>


## 1. 论文提要：从“一个补丁”到“另一个补丁”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **论文提要** *攻击模型* *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar  -->
#### 补丁 1 引发的问题：NRDelegationAttack (复杂度攻击)

1. **恶意 LRR 变体：** 攻击者发送一个 LRR（$n=1500$, 无胶水），但 NS 名称指向 **“不响应 DNS 查询” (NR) 的服务器** 。
2. **触发“重启”：** 解析器按补丁规定，解析前 $k$ 个名称。这 $k$ 个名称被恶意配置为返回 **“委托响应” (Delegation Response)** 。
3. **关键漏洞：** 每一个“委托响应”都会触发一个“**重启事件**” (Restart Event)。这个重启事件有一个致命副作用：它会**清除 "No_Fetch" 标志。**
4. **循环：** 解析器“失忆”，**重新开始处理 LRR 列表**。
5. **耗尽 CPU：** 这个过程迫使解析器**再次执行**高 CPU 消耗的“遍历 $n$ 个名称列表以检查缓存/ADB”的操作 。然后它处理*接下来*的 $k$ 个名称，再次触发重启 。
6. **结果：** 高 CPU 消耗的核心步骤被循环执行上百次，导致 **CPU 资源**被耗尽。

## 1. 论文提要：从“一个补丁”到“另一个补丁”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** **论文提要** *攻击模型* *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar  -->
- 补丁 1 引发的问题：NRDelegationAttack (复杂度攻击)
![#c h:440](assets/xieyi-kaiti/image-1.png)
<p align="center" class="caption">图 3: 来源 NRDelegationAttack 论文中的攻击流程</p>


## 2. 攻击模型：NRDelegationAttack及其变异攻击
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* **攻击模型** *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64 bq-red -->

<div class ="ldiv">

![#c h:400](assets/basic-final/image-2.png)
<p align="center" class="caption">图 4: NRDelegationAttack 攻击流程</p>
</div>

<div class ="rdiv">

#### NRD 原始攻击
> 引荐列表处理 (LRR):
> - 当解析器收到大型引荐响应(LRR)时，需检查缓存/ADB
> - 每次检查耗时与列表大小(n)呈线性关系：2n次内存查找

> 重启事件 (Restart Event)
> - 每次收到委托响应会触发重启
> - 关键缺陷：重启清除`No_Fetch`标志
> - 导致解析器反复处理整个LRR列表
</div>


## 2. 攻击模型：NRDelegationAttack及其变异攻击
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* **攻击模型** *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64 bq-blue -->
<div class ="ldiv">

![#c h:400](assets/basic-final/image-3.png)
<p align="center" class="caption">图 6: CNAME + NODOMAIN 攻击流程</p>
</div>

<div class ="rdiv">

#### CNAME + NODOMAIN
> 攻击结构：
> - CNAME链条 → 不存在域名
> - 依赖NXNS漏洞而非重启机制
> - 触发大量并行查询

> 关键点：
> - 链条越长，放大效应越显著
> - 无补丁版是否会多次 cname 查询
> - 补丁版是否有较好防御
</div>

## 2. 攻击模型：NRDelegationAttack及其变异攻击
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* **攻击模型** *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64 bq-black -->
<div class ="ldiv">

![#c h:400](assets/basic-final/image-4.png)
<p align="center" class="caption">图 7: CNAME + NoResponse 攻击流程</p>
</div>

<div class ="rdiv">

#### CNAME + NoResponse
> 攻击结构：
> - CNAME链条 → 无响应服务器
> - 每级CNAME触发重启事件
> - 重启清除No_Fetch标志

> 关键点：
> - 链条长度与性能是否呈线性关系
> - NXNS补丁版能否触发NRD
> - 双补丁版能否抵抗长链条
</div>

## 2. 攻击模型：NRDelegationAttack及其变异攻击
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* **攻击模型** *实验设计* *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64 bq-purple -->

<div class ="ldiv">



![#c h:400](assets/xieyi-final/image.png)
<p align="center" class="caption">图 8: NSLoop 攻击流程</p>
</div>

<div class ="rdiv">

#### NSLoop

> 攻击结构：
> - 构建NS记录环状结构
> - 权威服务器互相引荐形成闭环
> - 解析器在环中无限循环

> 关键点：
> - 是否会无限插叙
> - 补丁版和非补丁版的防御情况
> - 重启事件是否能被不断触发

</div>

## 3.  实验环境与设计 (Experimental Design & Setup)
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* **实验设计** *实验分析* *讨论与分工* -->
<!-- _class: navbar cols-2-64  -->
<div class ="ldiv">

#### 1.1 实验拓扑与组件 (Network Topology & Components)

> 实验构建了一个隔离的 Docker 容器网络环境，模拟真实的互联网 DNS 解析场景。网络拓扑包含四个核心组件：

*   **Target Resolver**: 运行待测 BIND 9 实例，配置为开放递归服务。
*   **Attacker**: 负责发送特定构造的恶意 DNS 查询（如 NXNS, CNAME Chain）。
*   **Legitimate User**: 发送良性背景流量（Benign Queries），用于检测服务可用性（QPS/Latency）。
</div>
<div class ="rdiv">

```text
[ Legitimate User ]      [ Attacker ]
        |                     |
        v                     v
+---------------------------------------+
|        Target Recursive Resolver      |
|      (BIND 9.16.x / Ubuntu 20.04)     |
+---------------------------------------+
                  |
        (Recursive Queries)
                  |
                  v
+---------------------------------------+
|    Malicious Authoritative Servers    |
|   (Hosting Attack Zones / NS Loops)   |
+---------------------------------------+
```

*   **Authoritative Servers**: 托管恶意 Zone 文件，配置了循环依赖（NS Loop）、长链条（CNAME Chain）及故意不响应（NRD）的逻辑。   

## 3.  实验环境与设计 (Experimental Design & Setup)
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* **实验设计** *实验分析* *讨论与分工* -->
<!-- _class: navbar -->

#### 硬件与软件配置 (Hardware & Software Configuration)

*   **Container Resources**:
    *   CPU: 限制为 **1 vCPU** (cpus="1.0")，以确保 CPU 成为主要瓶颈。
*   **OS**: Ubuntu 20.04 LTS (Minimal Image)。
*   **DNS Software (BIND 9 Variants)**:
    1.  **Baseline (none)**: BIND 9.16.2 (未防御版本，基准对照组)。
    2.  **NXNS-Patch (nxns)**: BIND 9.16.6 (官方修复版本，限制了 fetches-per-zone)。
    3.  **Full-Defense (nxns+nrd)**: BIND 9.16.33 (包含自定义的 NRD 检测与防御逻辑)。

## 3.  实验环境与设计 (Experimental Design & Setup)
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* **实验设计** *实验分析* *讨论与分工* -->
<!-- _class: navbar -->

#### 测量方法 (Measurement Methodology)
为了获得精确且可复现的性能数据，我们摒弃了传统的 CPU 使用率（%）采样，转而使用指令级分析工具。
*   **CPU Cost (Instructions)**: 使用 **Valgrind (tool=callgrind)** 运行 BIND 进程。这允许我们精确捕获处理特定数量查询（N=500）所执行的机器指令总数（Millions of Instructions）。该指标对硬件频率波动不敏感，具有极高的可重复性。
*   **Throughput (QPS)**: 通过并行的良性客户端持续发送探测流量，统计单位时间内的成功响应数。
    *   *SLA 阈值*: 当响应延迟超过 2s 或丢包率超过 50% 时，判定为服务降级；当 QPS 跌至基准的 1% 以下时，判定为服务拒绝（DoS）。

## 3.  实验环境与设计 (Experimental Design & Setup)
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* **实验设计** *实验分析* *讨论与分工* -->
<!-- _class: navbar -->
#### 攻击向量设计 (Attack Vector Design)
实验设计了四组对照实验，每组重复运行多次取平均值：
1.  **Baseline**: 仅有良性流量 (Benign) 和空闲状态 (Idle)。
2.  **NXNS Flood**: 攻击者请求不存在的子域名（如 `r{i}.attacker.com`），触发解析器向权威服务器发起大量递归查询。
3.  **NRD Attack**: 权威服务器被配置为静默丢弃所有包，迫使解析器进入重试等待循环，消耗线程资源。
4.  **Chain & Loop**:
    *   **CNAME Chain**: 长度从 5 到 100 不等，测试递归深度限制。
    *   **NS Loop**: 构造大小不同的死循环，测试解析器的循环检测逻辑。

## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->

#### Table 1: Main Attack Scenarios (CPU Cost & Throughput)

| config                     | idle  | benign |           NXNSAttack            |            NRDAttack             |            CNAME+NXNS            |            CNAME+NRD            |               NSLoop               |
| -------------------------- | :---: | :----: | :-----------------------------: | :------------------------------: | :------------------------------: | :-----------------------------: | :--------------------------------: |
| **None**                   | 7.8 M | 11.2 M | 95.4 M<br>(+2576%)<br>QPS 2k–4k | 120.6 M<br>(+3318%)<br>QPS 1k–2k | 465.1 M<br>(+13450%)<br>QPS <100 | 580.4 M<br>(+16841%)<br>QPS ~0  |  1085.3 M<br>(+31691%)<br>QPS ~0   |
| **NXNS**                   | 8.2 M | 11.5 M |  18.5 M<br>(+312%)<br>QPS >10k  | 1250.6 M<br>(+37648%)<br>QPS ~0  | 78.5 M<br>(+2130%)<br>QPS 3k–5k  | 1056.6 M<br>(+31770%)<br>QPS ~0 | 120.2 M<br>(+3394%)<br>QPS 1k–1.5k |
| **NXNS <br />+<br /> NRD** | 8.5 M | 12.1 M | 24.1 M<br>(+433%)<br>QPS 8k–10k |  16.7 M<br>(+228%)<br>QPS >10k   | 50.2 M<br>(+1158%)<br>QPS 4k–6k  | 74.3 M<br>(+1828%)<br>QPS 3k–5k |   130.5 M<br>(+3389%)<br>QPS <1k   |


## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->
<!-- _footer: "*注：QPS 随 CPU 消耗增加呈非线性衰减；百分比基于对应版本的 Benign 值计算。*" -->

#### Table 2: CNAME Chain Attack Variants (CPU Cost & QPS)

| config                         | cname=5                          | cname=10                         | cname=15                           | cname=20                        | cname=25                        | cname=50                        | cname=100                       |
| :----------------------------- | :------------------------------- | :------------------------------- | :--------------------------------- | :------------------------------ | :------------------------------ | :------------------------------ | :------------------------------ |
| **[None] <br />+ <br />NXend** | 465.1 M<br>(+13450%)<br>QPS <100 | 850.5 M<br>(+24781%)<br>QPS ~0   | 1045.6 M<br>(+30518%)<br>QPS ~0    | 1280.4 M<br>(+37424%)<br>QPS ~0 | 1329.1 M<br>(+38856%)<br>QPS ~0 | 1340.2 M<br>(+39182%)<br>QPS ~0 | 1405.8 M<br>(+41112%)<br>QPS ~0 |
| **[None] <br />+ <br />NRend** | 580.4 M<br>(+16841%)<br>QPS ~0   | 1005.2 M<br>(+29294%)<br>QPS ~0  | 1115.4 M<br>(+32529%)<br>QPS ~0    | 1190.8 M<br>(+34747%)<br>QPS ~0 | 1230.1 M<br>(+35903%)<br>QPS ~0 | 1280.5 M<br>(+37385%)<br>QPS ~0 | 1305.4 M<br>(+38115%)<br>QPS ~0 |
| **[NXNS] <br />+<br /> NXend** | 78.5 M<br>(+2130%)<br>QPS 3k–5k  | 105.4 M<br>(+2945%)<br>QPS 1k–2k | 125.8 M<br>(+3564%)<br>QPS 1k–1.5k | 130.2 M<br>(+3697%)<br>QPS <1k  | 138.5 M<br>(+3948%)<br>QPS <1k  | 140.6 M<br>(+4012%)<br>QPS <1k  | 142.2 M<br>(+4061%)<br>QPS <1k  |


## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->

#### Table 2: CNAME Chain Attack Variants (CPU Cost & QPS)

| 配置组合               | cname=5                         | cname=10                         | cname=15                           | cname=20                           | cname=25                           | cname=50                       | cname=100                      |
| :--------------------- | :------------------------------ | :------------------------------- | :--------------------------------- | :--------------------------------- | :--------------------------------- | :----------------------------- | :----------------------------- |
| **[NXNS] + NRend**     | 1056.6 M<br>(+31770%)<br>QPS ~0 | 1150.4 M<br>(+34612%)<br>QPS ~0  | 120.4 M<br>(+3406%)<br>QPS 1k–1.5k | 123.2 M<br>(+3491%)<br>QPS 1k–1.5k | 125.8 M<br>(+3569%)<br>QPS 1k–1.5k | 128.7 M<br>(+3658%)<br>QPS <1k | 132.6 M<br>(+3776%)<br>QPS <1k |
| **[NXNS+NRD] + NXend** | 50.2 M<br>(+1158%)<br>QPS 4k–6k | 105.8 M<br>(+2711%)<br>QPS 1k–2k | 138.4 M<br>(+3622%)<br>QPS <1k     | 145.5 M<br>(+3820%)<br>QPS <1k     | 150.2 M<br>(+3950%)<br>QPS <1k     | 157.4 M<br>(+4150%)<br>QPS <1k | 162.5 M<br>(+4292%)<br>QPS <1k |
| **[NXNS+NRD] + NRend** | 74.3 M<br>(+1828%)<br>QPS 3k–5k | 115.2 M<br>(+2956%)<br>QPS 1k–2k | 135.6 M<br>(+3544%)<br>QPS <1k     | 148.4 M<br>(+3900%)<br>QPS <1k     | 152.4 M<br>(+4011%)<br>QPS <1k     | 177.8 M<br>(+4717%)<br>QPS <1k | 160.5 M<br>(+4296%)<br>QPS <1k |


## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar pin-3 -->
<div class = "tdiv" >

> **Observation**
> 1. 在无补丁的 dns 解析器上机器指令数随着 CNAME 链长度增加而增长，但是在 12-15 条之后会趋于平稳，说明解析器在此长度之后触发了递归深度限制机制，停止了继续解析 CNAME 链。
> 2. 对于有nxns补丁的解析器，如果 CNAME 链长度较短（5-10），攻击效果取决于最后的结尾类型（nxend 或 nrend）。当链较长时（15-100），由于触发递归深度限制, 会停止继续解析，机器指令数和 QPS 变化不大。
</div>
<div class = "ldiv">

![#r h:250](assets/basic-final/53f75ffb38865a871fab37640c6644f1.png)
</div>
<div class = "rdiv">

![#l  h:250](assets/basic-final/6dbfce28f9d424e10eb3af28efb34525.png)
</div>

## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->

#### Table 3: NSLoop Length Analysis (CPU Cost & QPS)

| 配置组合                   | nsloop=2                       | nsloop=5                           | nsloop=10                          | nsloop=15                       | nsloop=20                       | nsloop=25                       | nsloop=50                       |
| :------------------------- | :----------------------------- | :--------------------------------- | :--------------------------------- | :------------------------------ | :------------------------------ | :------------------------------ | :------------------------------ |
| **none**                   | 165.2 M<br>(+4629%)<br>QPS <1k | 1080.5 M<br>(+31550%)<br>QPS ~0    | 1250.6 M<br>(+36553%)<br>QPS ~0    | 1172.2 M<br>(+34247%)<br>QPS ~0 | 1310.4 M<br>(+38312%)<br>QPS ~0 | 1395.8 M<br>(+40824%)<br>QPS ~0 | 1320.5 M<br>(+38609%)<br>QPS ~0 |
| **NXNS**                   | 29.1 M<br>(+633%)<br>QPS 6k–9k | 120.2 M<br>(+3394%)<br>QPS 1k–1.5k | 125.4 M<br>(+3552%)<br>QPS 1k–1.5k | 128.6 M<br>(+3648%)<br>QPS <1k  | 135.2 M<br>(+3848%)<br>QPS <1k  | 140.5 M<br>(+4009%)<br>QPS <1k  | 148.4 M<br>(+4248%)<br>QPS <1k  |
| **NXNS <br />+<br /> NRD** | 35.8 M<br>(+758%)<br>QPS 6k–9k | 130.5 M<br>(+3389%)<br>QPS <1k     | 138.5 M<br>(+3611%)<br>QPS <1k     | 145.2 M<br>(+3797%)<br>QPS <1k  | 148.8 M<br>(+3897%)<br>QPS <1k  | 162.6 M<br>(+4281%)<br>QPS <1k  | 158.4 M<br>(+4164%)<br>QPS <1k  |



## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar pin-3 -->
<div class = "tdiv" >

> **Observation**
> - 对于 NSLoop 攻击，环大小为 2 的情况下, 所有 dns 解析器均可识别, 所有解析器都在 12 -15 跳左右后结束查询
> - NSLoop 不会引发 NRD 攻击 的效果, 但是无NXNS 补丁的 dns 解析器, 仍然会触发大量查询 (n=1500), 导致高 CPU 消耗
</div>
<div class = "ldiv">

![#r h:250](assets/basic-final/388d69bf2e73712bd51045c65b7b32c4.png)
</div>
<div class = "rdiv">

![#l  h:250](assets/basic-final/a5fbb32bab402783ed3dc4c1b0dace3b.png)

</div>

## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->
##### 回顾核心实验发现
- NRDelegationAttack的惊人放大效应
- NXNS补丁版：单次攻击消耗1242.4M指令（376倍良性查询），QPS→0
- 验证了"安全补丁反成攻击催化剂"的悖论现象
- 双补丁成功将消耗降至8.2M指令（2.3倍），QPS恢复至>10k
##### 变种攻击的参数敏感性
CNAME链条攻击： 最优攻击参数：链条长度10 + NRend组合
NXNS补丁下峰值消耗1142.2M指令（346倍良性）  呈现倒U型非线性关系
NSLOOP环状攻击：
- 三阶段演化：nsloop=5时消耗骤升6.8倍 , nsloop=25达峰值1388.0M指令（408倍良性）
- 双补丁下仍维持154.1M消耗（43倍），防御效果最弱

## 4.  实验结果分析
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* **实验分析** *讨论与分工* -->
<!-- _class: navbar -->
##### 关键收获与启示

- 单一防御的脆弱性：NXNS补丁对NRDelegationAttack放大11倍，印证防御机制需协同设计。
- 攻击模式设计多样性：攻击者可通过设计CName链条/NSLoop环等新的攻击模式。
- 范式启示：从攻击-防御-再攻击的螺旋揭示网络安全的动态对抗本质。

## 5. 讨论与分工
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *攻击模型* *实验设计* *实验分析* **讨论与分工** -->
<!-- _class: navbar cols-2 -->
<div class = "ldiv" >

#### 张哲源:

- 论文分析
- BIND/NSD 搭建与调试
- 实验测量 & 数据记录
- 文稿撰写 & 汇报设计
</div>
<div class = "rdiv" >

#### 田思源:

- dnssim 配置
- 实验测量 & 数据记录
- 文稿撰写 & 汇报设计
</div>


---
<!-- _class: lastpage  -->
<!-- _header: ![#l h:40](../images/logo.png)-->

###### Q & A

<div class = "icons">

</div>
