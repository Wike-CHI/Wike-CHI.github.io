---
title: "Timao重构手记：干掉Redis、引入Agent架构、用sherpa-onnx替换FunASR"
date: 2026-04-14
draft: false
tags: ["重构", "Agent", "AI", "直播系统"]
categories: ["技术实践"]
description: "一个直播助手从原型到产品的重构全过程。移除了什么，加入了什么，为什么这么干。"
---

第一版Timao能跑。但能跑和能用是两回事。

Redis、MySQL、用户认证、订阅支付系统——这些东西对于一个本地桌面应用来说，全是负担。

于是重写。

## 第一刀：砍掉Redis

原来用Redis做弹幕缓存、AI分析缓存、会话存储。部署的时候用户得先装Redis。

对于一个桌面应用，这很荒谬。

弹幕本身就是流式数据，过了就过了，不需要持久化缓存。会话数据用内存字典够了。AI分析结果直接推到前端，不存中间态。

RedisManager砍到只剩一个纯内存实现。整个Redis依赖链全部移除。

部署门槛从"装Python + Node.js + Redis + MySQL"变成了"装Python + Node.js"。

## 第二刀：MySQL换成SQLite

同理。一个本地工具，不需要MySQL。

SQLite开WAL模式，并发读写完全够用。数据就是一个文件，拷贝就能迁移，备份就是复制。

数据库初始化脚本写好，第一次启动自动建表。用户不需要手动操作任何东西。

## 第三刀：移除用户认证和订阅系统

早期设计里有一套完整的用户系统——注册、登录、订阅、支付。因为想做成SaaS。

后来想清楚了：直播数据不能上云，这个产品就该是本地的。

砍掉用户认证、砍掉订阅、砍掉支付。前端相关的UI全部清理。应用打开就能用，不需要登录任何东西。

代码少了，复杂度低了，用户不用关心"我买了什么套餐"。

## 加入：Agent架构

砍完了旧的，加新的。

原来的AI分析是单体函数调用。弹幕来了，调一次AI，出结果。没有上下文，没有多步推理，没有反馈。

重构后引入Agent体系：

**BaseAgent** — 所有Agent的基类，Pydantic AI兼容。统一输入输出格式，自动计时，自动错误处理。

**DanmakuAgent** — 专门处理弹幕。过滤噪声、识别关键互动、判断哪些弹幕值得主播关注。

**VoiceAgent** — 语音转写的Agent层。支持多ASR后端切换（SenseVoice、sherpa-onnx、讯飞），运行时可以切，不用重启。

**AnalyzerAgent** — 高速分析，支持MiniMax做快速判断。不需要深度推理的场景用轻量模型，省时间。

**DecisionAgent** — 决策Agent，接GLM-5的思考模式。需要深度分析的场景（比如直播节奏判断、话术推荐）用大模型。

**ReflectionAgent** — 反思Agent，评估其他Agent的输出质量。分析结果先过一遍反思，不靠谱的就重新来。

所有Agent通过 `workflow_v2` 编排。不是串行调用，是按需调度。

## 加入：AI Gateway 2.0

原来调AI就是直接用LangChain的链式调用。一个模型打天下。

问题很明显：不同任务需要不同模型。

弹幕分析要快，用MiniMax。话术生成要好，用GLM-5。简单判断用轻量模型省钱。

AI Gateway 2.0做的事情：

- **智能路由**：根据任务类型自动选模型
- **思考模式**：GLM-5的深度推理开关，复杂任务开启，简单任务关闭
- **流式输出**：chat_completion走流式，前端实时显示AI生成过程
- **多服务商**：智谱GLM、MiniMax、讯飞、豆包、DeepSeek，配置里加一行就能用

## 加入：sherpa-onnx替换FunASR

SenseVoice模型之前用FunASR加载。FunASR依赖PyTorch，光PyTorch就几个G。

sherpa-onnx是纯ONNX推理，不需要PyTorch。模型加载快，内存占用低，推理速度不差。

迁移之后安装依赖从好几个G砍到几百MB。对于桌面应用的分发来说，这是质的区别。

语音转写层做了抽象——`transcriber`接口统一，底层是FunASR还是sherpa-onnx，上层代码不用关心。

## UI也重构了

暗色主题从冷色调换成了暖琥珀色。移除了所有emoji，换成Lucide图标系统。卡片动画加了弹性过渡。

不是因为好看。是因为主播在直播间盯着屏幕几个小时，冷色调的UI会让人疲惫。暖色更耐看。

悬浮窗从内嵌改成了独立窗口。可以拖到副屏，不占主界面空间。

## 重构前 vs 重构后

| | 重构前 | 重构后 |
|---|---|---|
| 数据库 | MySQL | SQLite |
| 缓存 | Redis | 内存 |
| ASR | FunASR (PyTorch) | sherpa-onnx |
| AI调用 | 单模型链式 | Agent + Gateway路由 |
| 用户系统 | 注册/登录/订阅 | 无，打开即用 |
| 部署依赖 | Python+Node+Redis+MySQL | Python+Node |

代码在 [GitHub](https://github.com/Wike-CHI/timao-douyin-live-manager)，欢迎提issue。
