+++
title = "关于aPaaS平台的思考"
date = 2024-03-20T19:11:14+08:00
lastmod = 2024-03-20T19:11:14+08:00
author = ["jiandong.liu93"]
draft = true
tags = ["aPaaS"]
+++

# 什么是Builder

Builder scope划分我的认知里，是从前端视角产生的，基于组件编辑产生DSL，并且解析DSL产出页面。

从一个更中立的视角来看，其实这是两件事，IDE与运行容器。而这两者之间，隐含了一个不被重视的服务端工作，就是DSL的持久化与维护。

# 产品演进

- 前身是kunlun
- 我们是服务B端的，少数大客户的边缘Case也是要处理的，或者说很重要
- 我们曾经在两个路径里摇摆，服务字节还是搞好商业化
- 这一年的主题，删繁就简
- 支持了电商，但又抛弃了电商
- 6月接客

# 前端演进

- 1.0 2.0 3.0

- 没法纠结技术细节，前端内容不是很懂，投入研究的ROI太低了。

# 技术现状

> 由于之前一直没有服务端团队，所以所有的资料都是从前端视角来做的。尽快完成一份服务端的梳理与规划文档十分关键

## 设计态

> 核心问题，一个页面有哪些信息来描述，保存在哪里
> 
> 从PageConfig的存在，可以初步总结Builder的服务端在这里有两个核心职责：
> 
> 1. 数据的持久化，未来的协同编辑，增量存储等
> 
> 2. IDE的体验支持（既然叫IDE，有什么是本地解决不了的，我在把自己越做越薄？）

基于现有的资料：

页面=基本信息+UIDL+Page Config

1. 基本信息：页面的标题描述等

2. UIDL：对页面内容的定义

3. Page Config：**服务端需要感知的内容**，导航信息，权限树，多语言，环境变量等。

我的问题：

1. 这个划分逻辑是否合理，Page Config看起来更像是Index，用来辅助IDE运行提示。是否应该影响页面本身的定义与组织

2. 写入时机是否合理，page config的实时性要求是什么样的。检索UIDL生产page config保存的设计是否合理。

3. 如果UIDL只能全量保存，就不应当支持并发编辑。进一步的page config的开发辅助功能可以本地进行。为什么要向服务端投递。

4. Builder DSL希望在一份文件内包含页面所有的表达，实质上已经弱化了Page Config的功能，一个IDE的Index使用应当影响主流程性能。应当向旁路转移

但以上更多的是基于存储视角的逻辑，设计态应该进行什么样的表达，元素有哪些没有进行完整的讨论。

1. 类型

2. 位置 或者说 层级

3. 属性

4. 行为 这里包含 数据关联

## “编译”

> 在设计态完成设计，与运行态执行之间，服务端可以做什么
> 
> 这个是我们可以摆脱前端做的，或者说前端做不了的。在这两个环节之间，前端没有任何实体。
> 
> Builder DSL的“编译期优化”
> 
> 时效性上，当前的产品设计上，发布流程会给服务端多少时间与空间。
> 
> 服务端直接渲染出HTML+JS的代码不是更强，有没有办法做0成本抽象

必须找到实际可解决的问题。好好深入一下运行态的问题。

1. ByteFX编译耗时很慢，目前被前置到了设计态触发。这部分算力成本的转移是否合理呢

## 运行态

> 通过下发设计态的产物，让App跑起来

## 阶段目标

1. 明确业务边界

2. 我们在做什么，要做什么

3. Prove我们自己的价值，之前只有前端也能运转，我们7个的价值是什么

## 关联业务方

> 谁为我们提供了什么样的支持，我们之间的边界是啥

1. 各种Gateway

2. RefService：metadata服务中的另一个模块，mt_ref表中记录了页面对元数据的引用关系

3. Data

4. Rule

## 代码理解

1. 

# 远景展望

分离设计态运行时的支持信息，index独立维护管理，优化性能，甜点功能从主线分离。同时分离的分析可以更好的融合ai，做低代码的copilot

设计态运行态之间插入一个编译产物管理。这是前端视角缺失的内容，离开浏览器我们能做什么

目前我们在推广，卖点是研发效率，但“大销售期”度过后，被征服的开发者会将低码开发的产品投向C端（相对的C）使用，必然会产生运行态C端体验的优化，尽早布局可以提升天花板。

我们应该更重视编译产物的流转管理，持久化是builder服务端的本源定位。这也是我们的存在价值，提供灰度，ab test等更多能力。

编译期是用户无感且有明确预期的一个阶段。可以承担大量的计算成本。

涉及产物与运行产物的分离，这个收益如何论证，强行一直看起来天花板很低，需要前端帮忙论证。

加了编译期，搞lynx，抖音小程序啥的，都有的玩

那么有三个域，设计，支撑，分发。

设计，增量更新，并发编辑，编译优化。专注于保存设计产物，不要再设计期做过度计算

支撑，索引，**ai推荐（我真的觉得有的玩）**，优化ide性能。各种辅助与ChangeLog。

分发，编译产物管理，发布，灰度，运行期权限，数据拉取等能力，主要是运行态支持。未来的aPaaS产品的场景扩展性保障

安全问题，我们提供的能力是否有注入或者其他风险。弄挖矿代码进去怎么算

# 架构展望

> 以上的纯纯是业务理解，那么进一步的怎么去通过业务理解，转换成未来的架构设计。

# 杂项

- 如何论证收益，很关键
1. 耗时优化做到端到端分析，明确价值
- 浏览器限制，比如请求的排队。之前忽略了client的情况。一个之前没讨论过的topic

- 小程序首屏优化有没有啥积累

- UIDL是否是可公开的

- 网关每次的通过耗时。一个优化点
  
  计算环节有哪些是应该分配到服务端的，我们作为IDE还能做什么
  
  协同编辑的OT算法？只有服务端可以做
  
  本地执行？Or 飞书服务器托管，各自什么逻辑，支不支持开发者植入登录态，还是我们自己支持的。如果是，登录态校验逻辑我们包给开发者？
  
  1. 如果我们做。我们跟data和auto是啥关系