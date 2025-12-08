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

<!-- _class: cover_e -->
<!-- _paginate: "" -->
<!-- _footer: 互联网体系结构及其安全基础 -->

# <!-- fit -->基于 [USENIX Security '23] NRDelegationAttack 的研究复现 DNS 复杂度攻击

###### NRDelegationAttack: Complexity DDoS attack on DNS Recursive Resolvers
Reporter ：田思源 张哲源
Date ：2025 年 12 月 31 日

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
<!-- _class: navbar cols-2-73 bq-red -->

<div class ="ldiv">

![#c h:450](assets/basic-final/image-2.png)
</div>

<div class ="rdiv">

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
<!-- _class: navbar cols-2-73 bq-red -->


## 3.严重性量化：从“洪水”到“CPU 放大 5600 倍”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* **严重性量化** *研究计划* *讨论与分工* -->
<!-- _class: navbar cols-2-46 -->

<div class ="ldiv">

- **内部复杂度 (图 3)：**
  
  - 使用 `Valgrind` 工具测量，一次 NRDelegationAttack 恶意查询（$n=1500$）在打了补丁的 BIND9 上消耗了 **34 亿**条机器指令 。
  - 相比之下，一个良性查询仅消耗 **19.5 万**条指令 。
  - **结论：** CPU 复杂度**放大了 5600 倍以上** 。
  
</div>

<div class = "rdiv">

![alt text](assets/xieyi-kaiti/image-3.png)
</div>

## 3.严重性量化：从“洪水”到“CPU 放大 5600 倍”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* **严重性量化** *研究计划* *讨论与分工* -->
<!-- _class: navbar -->



**性能影响 (图 5)：**

- 在云环境中，NRDelegationAttack 导致的良性用户吞吐量 (QPS) **下降幅度远超 NXNSAttack** 
  
![#c h:500](assets/xieyi-kaiti/image-4.png)

## 3.严重性量化：从“洪水”到“CPU 放大 5600 倍”
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* **严重性量化** *研究计划* *讨论与分工* -->
<!-- _class: navbar cols-2-64 -->
<div class="ldiv">

![#c h:530](assets/xieyi-kaiti/image-5.png)
</div>

<div class ="ridv">

- **真实世界 (表 2)：**
  - 即使是“削弱版”攻击（$n=20$）：
  - 8.8.8.8 (Google) 延迟从 74ms 飙升至 **4700ms** 。
  - 1.1.1.1 (Cloudflare) 延迟从 80ms 飙升至 **5350ms** 。
  - VeriSign, Dyn, Yandex 等多个主流解析器**完全超时**。

</div>


## 4. 研究计划：复现与验证
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* *严重性量化* **研究计划** *讨论与分工* -->
<!-- _class: navbar -->

- **研究目标：** 在隔离环境中，复现 NRDelegationAttack，并量化其对 BIND9 解析器造成的“复杂度”和“性能”影响。
- **复现环境：** 使用论文作者提供的官方模拟器：
  - **`dnssim` (Inner-Emulator)** 。这是一个基于 Docker 的 DNS 隔离实验室，包含了所有必要的组件。
- **核心组件：**
  - **受害者：** BIND9 9.16.6（论文中测试的 NXNS-patched 易受攻击版本）。
  - **攻击者：** 本地配置的 NSD（Name Server Daemon）服务器，用于模拟：
    1. 一个 `referral.com` 服务器（用于发送 LRR）。
    2. 一个 `delegation.com` 服务器（用于发送“委托-无响应”响应链）。


## 4. 研究计划：复现与验证
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* *严重性量化* **研究计划** *讨论与分工* -->
<!-- _class: navbar cols-2 -->
 
 <div class = "ldiv">

**复现指标 (MOMs)：**

  1. **CPU 复杂度 (对标图 3)：**

     - **工具：** `Valgrind` 。
     - **指标：** 测量在不同 $n$ 值（例如 $n=100, 500, 1500$）下，单次恶意查询消耗的**机器指令数** 。

  2. **吞吐量 (对标图 5)：**

     - **工具：** `Resperf` 。

     - **指标：** 测量在持续攻击下，“良性”客户端的**QPS 吞吐量**下降情况 。
  </div>

<div class = "rdiv">

![#c h:250](assets/xieyi-kaiti/image-6.png)
![#c h:250](assets/xieyi-kaiti/image-7.png)
</div>

## 5. 讨论与分工
<!-- _header: \ ***![#l h:40](../images/logo.png)*** *研究背景* *核心漏洞* *严重性量化* **研究计划** *讨论与分工* -->
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
