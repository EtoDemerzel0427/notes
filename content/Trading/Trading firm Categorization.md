---
title: Trading firm Categorization
slug: trading-firm-categorization
date: 2025-12-19
tags: [quant]
category: Trading
draft: false
---

互联网上提到trading firm的时候，人们常常提到一些名词：
market maker、hedge fund、prop shop、sell side、buy side、HFT，etc。这些概念曾一度让我十分困惑。其实他们来自的是不同的分类体系，回答的也是完全不同的问题。

## 一、核心分类逻辑
金融机构的本质差异主要由以下两个维度的组合决定：

* 资金来源：界定机构管理资金的法律属性与责任边界。
* 收益模式：界定机构在市场微观结构中的角色与风险承担方式。
* 市场生态角色：界定机构在市场生态中所处的位置，以及其服务对象。
* 时间尺度：机构在交易决策、报价更新和风险调整中所采用的典型时间尺度。

## 二、第一维度：资金来源（Capital Structure）

这个维度回答的是一个最基础的问题：
> 交易风险最终由谁承担？

### 1. 自有资金（Proprietary Capital / Prop）

使用的是公司自己的资产负债表进行交易。
在这种结构下：

* 没有外部投资人（LP）
* 不存在受托责任（fiduciary duty）
* 盈亏完全由公司内部消化
* 策略、风险和回撤不需要对外解释

“Prop shop / proprietary trading firm”指的仅仅是这一点，并不隐含任何具体交易风格。

### 2. 客户资金（Client / LP Capital）

使用的是外部投资人的资金进行交易和投资。在这种结构下：

* 机构对 LP 负有法律意义上的受托责任
* 必须解释风险来源、回撤和收益结构
* 受到更严格的合规、披露和监管要求

传统意义上的 hedge fund、asset manager 都属于这一类。

**结论：**
Prop vs Client 是一个资金与法律关系的分类，不是交易模式分类。

## 三、第二维度：交易与风险模式（Trading / Risk Model）

这个维度回答的是另一个问题：
> 钱是通过承担什么样的风险赚到的？

### 1. 做市 / 流动性提供（Market Making / Liquidity Providing）

做市是一种市场功能性角色。其核心特征是：
* 持续在市场上报买价和卖价
* 接收并处理订单流
* 为他人交易提供即时对手方

在理想状态下，做市商并不希望承担方向性风险 (delta-neutral)：
* 不押注长期涨跌
* 不追求市场 alpha
* 持仓更多是库存（inventory），而非观点表达

做市的盈利来自于：
* 买卖价差（spread）
* 流动性补偿
* 定价、执行和风险管理能力

### 2. 方向性 / Alpha 投资（Directional / Alpha Investing）

这是“投资”在金融语境中的典型含义。其核心特征是：
* 主动承担价格、相关性或波动风险
* 明确押注某种判断或模型
* 希望获得超出市场本身的回报（alpha）

在这种模式下：

* 长期或中期 exposure 是刻意为之的
* 回撤是策略的一部分
* 收益高度依赖判断是否正确

**结论：**
Market making 与 alpha investing 的区别不在于“频率高不高”，
而在于是否主动、持续地暴露在方向性风险之中。

## 四、第三维度：市场生态角色（Sell Side vs Buy Side）

这是最容易被误解、但又极其重要的维度。它回答的是：

> 你在为谁服务？你在市场里“卖”的是什么？

### 1. Sell Side（卖方）

Sell side 指的是那些向市场其他参与者提供服务和产品的机构。这里“卖”的不是股票本身，而是：

* 流动性
* 报价
* 执行能力
* 清算、融资、借贷
* 研究或结构化产品

典型 sell side 机构包括：

* 投行的交易与做市部门
* 独立做市商和高频做市公司
* 经纪商、prime broker

做市商在本质上是**纯 sell side**：
它们存在的意义就是为他人交易提供对手方。

### 2. Buy Side（买方）

Buy side 指的是那些把资金配置到资产中、并最终承担投资风险的机构。它们：

* 不向市场提供流动性作为主要业务
* 使用 sell side 提供的服务进行交易
* 承担价格变化带来的最终盈亏

典型 buy side 包括：
* Hedge fund
* Mutual fund （公募基金）
* Pension / endowment （养老金/捐赠基金）
* Family office （家族办公室，通常为超级富豪家族工作）

**结论：**
Sell side / Buy side 描述的是你在金融生态中的位置和功能，
而不是资金来源或交易频率。

## 五、第四维度：时间尺度 / 交易频率（Time Horizon / Trading Frequency）
这是一个重要但从属的补充维度。

它回答的是：
> 交易决策和持仓通常发生在什么时间尺度上？

“高频 / 低频”描述的是实现方式，而不是：

* 资金结构
* 经济角色
* 风险本质

因此，它不能替代前面的任何一个维度，但可以在它们之上叠加。

常见的时间尺度划分（非严格边界）

* 高频（HFT）：微秒到秒级
* 中频（MFT）：分钟到天级
* 低频（LFT）：周到年级

这是一个连续谱，而不是离散分类。

**关键澄清**:

* 高频不等于做市
* 低频不等于对冲基金
* 同一家机构、甚至同一策略，可以横跨多个时间尺度
* 频率往往是约束条件的结果，而不是商业定位的起点
* “高频”在不同资产类别中并不意味着同样的时间要求，如股票，因为决策维度单一，其高频的时间量级通常在纳秒到微秒之间；而对于期权，由于它是一个由多个维度共同决定的合约集合，需要在高位风险上进行复杂的一致性管理，其高频量级通常是微秒到毫秒级别。

## 六、四个维度合在一起的完整视图

| 机构                 | 资金来源 | 风险模式                | 市场角色      | 时间尺度 |
| ------------------ | ---- | ------------------- | --------- | ---- |
| Akuna Capital      | 自有   | Market Making       | Sell Side | 中–高  |
| Citadel Securities | 自有   | Market Making       | Sell Side | 高    |
| Jane Street        | 自有   | Market Making       | Sell Side | 中–高  |
| IMC / Optiver      | 自有   | Market Making       | Sell Side | 高    |
| Citadel（HF）        | 客户   | Alpha Investing     | Buy Side  | 中    |
| Renaissance        | 客户   | Quant Alpha         | Buy Side  | 高–中  |
| Bridgewater        | 客户   | Macro / Directional | Buy Side  | 低    |

