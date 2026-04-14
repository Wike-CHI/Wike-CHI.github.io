---
title: "用Tauri替代Electron做图像工具，打包体积从200MB砍到8MB"
date: 2026-04-14
draft: false
tags: ["Tauri", "Electron", "图像处理", "性能"]
categories: ["技术对比"]
description: "同一个AI抠图工具，分别用Electron和Tauri实现后的真实对比。不只是体积差距。"
---

做AI抠图工具的时候，先出的Electron版本。能跑，但总觉得自己在用大炮打蚊子。

一个抠图工具，打包出来200MB。里面塞着一个完整的Chromium。

用户下载的时候会想：我就是要个抠图啊。

## 换Tauri

Tauri 2.0出来之后，用Rust替掉了Chromium，前端还是用的Web技术。

同一个功能——AI智能抠图、白底图生成、批量处理——重写了一遍。

打包体积：8MB。

没有写错。两百兆变八兆。

用户不需要装任何运行时。双击打开直接用。

## 前端几乎不用改

React写的界面，从Electron搬到Tauri，改动很小。

Tauri的API设计和Electron有点不一样，但思路差不多。前端通过 `@tauri-apps/api` 调用后端能力。

唯一要适应的是Rust那层。如果你的后端逻辑原来就在Python里（比如我这边的AI推理），可以通过Tauri的sidecar或者command调外部进程。

我的做法是Tauri壳子负责UI，FastAPI还是跑在本地做推理。和Electron那套架构类似，只是壳子轻了很多。

## 性能差距在实际使用中感受明显

Electron版本的冷启动要3-4秒。Chromium要加载，渲染进程要初始化。

Tauri版本冷启动不到1秒。

批量处理100张图片的时候，Tauri版本的内存占用稳定在150MB左右。Electron版本起步就300MB，处理到后面能飙到500MB。

原因很简单：Electron带了整个浏览器引擎，Tauri用的是系统自带的WebView。

## 但Tauri也有坑

**兼容性。** Windows上依赖WebView2（Edge内核），老一点的Windows 10可能需要手动装。Electron自带Chromium，不依赖系统环境。

**生态差距。** Electron的npm生态非常成熟，遇到问题基本都能搜到。Tauri还在成长中，有些edge case只能翻文档或者看源码。

**调试体验。** Electron的DevTools就是Chrome DevTools，非常顺手。Tauri在Windows上也能开DevTools，但偶尔有些奇怪的渲染差异需要系统WebView背锅。

## 我的结论

如果你的应用是——

- 重前端、轻后端（管理工具、笔记应用、小工具）
- 需要小体积、快启动
- 用户群技术素养还行（能接受装WebView2）

→ **Tauri，毫不犹豫。**

如果你的应用是——

- 需要兼容各种老系统
- 大量依赖Electron特有生态（比如某些npm包直接调Node API）
- 团队全是JS/TS，不想碰Rust

→ **Electron还是更稳。**

我的抠图工具最后选了Tauri。8MB的安装包，跨平台，用户反馈很好。

项目在 [GitHub](https://github.com/Wike-CHI/aiimage12334)，可以看看实现细节。
