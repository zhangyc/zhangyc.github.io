---
title: Flutter add-to-app ProMotion 拖影的底层机制与解决思路
date: 2026-02-03
tags:
  - Flutter
  - iOS
  - add-to-app
  - ProMotion
  - 性能
---

## 现象与复现

在 iPhone 17 系列（120Hz ProMotion 设备）中，Flutter add-to-app 页面嵌入到原生 Swift 容器后，横向或纵向滑动时会出现明显“拖影/残影”。同一页面在纯 Flutter App 中则不明显或无法复现。

**关键排查细节：**
通过性能工具分析发现，**Flutter 的每一帧渲染耗时极短（<2ms）**。这直接排除了“Flutter 性能不足导致掉帧”的可能性。既然渲染这么快，为什么肉眼还是看到了拖影？

## 一句话结论

问题的根因是 **Flutter 渲染节奏与原生容器的显示刷新节奏不一致**，导致 GPU 合成时出现帧间不齐，体现为拖影/重影。仅在 `Info.plist` 声明 120Hz 只是修复了“症状”，而不是解释了“机制”。

## 机制拆解：为什么频率不一致会“拖影”？

下面用更“底层”的视角来拆解：

### 1. iOS 刷新节奏与 ProMotion

ProMotion 设备的屏幕刷新率是动态的：系统会在 60/80/90/120Hz 之间切换，以节能为优先目标。**刷新率由系统与内容一起协同决定**，而不是应用自己强制固定。

- 原生视图（UIKit/SwiftUI）通过 `CADisplayLink` 与系统刷新同步。
- 系统根据当前内容、触摸输入、滚动状态、视频播放等因素决定实际刷新率。

### 2. Flutter 的渲染模型

Flutter 在 iOS 上使用 **Raster + Skia + Metal**，运行在独立的渲染管线：

- 引擎维护自己的 `CADisplayLink`，用于触发每帧的 `beginFrame`。
- 如果系统认为 Flutter 页面“只需 60Hz”，Flutter 可能就以 60Hz 产生帧。
- 原生容器如果在 120Hz 的滚动动画中合成 Flutter 视图，就会出现帧间不对齐。

### 3. add-to-app 的“节奏分裂”

在 add-to-app 中，**Flutter 不是主视图**，而是作为一个子视图嵌到原生视图层级中：

- 原生视图滚动依然由系统以 120Hz 驱动。
- Flutter 视图内容可能仍在 60Hz 出帧。
- 合成时，就会出现“同一帧中原生已经前进，Flutter 还停留在上一帧”的视觉拖影。

> 这种现象在纯 Flutter App 中不明显，是因为 **Flutter 与整个 App 的节奏更容易保持一致**，系统会倾向于给它完整的刷新预算。

### 4. “快”不等于“同步”

这也完美解释了为什么**单帧渲染小于 2ms 依然会拖影**：

即使 Flutter 2ms 就画完了这一帧，但如果它是按照 60Hz 的节奏（每 16.6ms）提交给系统的，而原生滚动容器是按照 120Hz（每 8.3ms）刷新的。

那么在原生滚动的中间那一帧（第 8.3ms 时），Flutter 的新帧虽然早就画完了（因为只花了 2ms），但它可能还没有“提交”或者系统认为它还没到显示时间（Presentation Time）。

结果就是：**原生容器动了，Flutter 画面没动**。等到下一个 16.6ms 时，Flutter 画面才跳变。这种“一帧动、一帧不动”的交替，在人眼看来就是严重的抖动和残影。

## 为什么 `Info.plist` 声明 120Hz 会“解决”

在 `Info.plist` 里声明 `CADisableMinimumFrameDurationOnPhone` 或相关配置，本质上是告诉系统 **Flutter 视图也具备高帧率刷新能力**，从而让系统愿意将高刷新预算分配给该视图。

这样 Flutter 的 `CADisplayLink` 就能更频繁触发：

- Flutter 出帧节奏上升
- 与原生滚动合成时帧间差距缩小
- 拖影消失

> 但这只是“强行拉齐频率”，并没有解释它背后的节奏差异为何产生。

## 更深一层：时钟域与合成阶段

可以把 Flutter 和 UIKit 看成两个“时钟域”：

- **UIKit/SwiftUI**：跟随系统显示刷新
- **Flutter Engine**：跟随自己的 `CADisplayLink`

这两个时钟域不同步时，会导致合成阶段出现“半帧延迟”。

在 iOS 的合成 pipeline 中，系统会在一个固定的 vsync 时刻对所有视图进行最终合成。如果 Flutter 的帧没有及时准备好，就只能用上一帧的纹理参与合成，从而出现拖影感。

## 实践建议（不仅仅是改 plist）

1. **确保 FlutterView 的绘制节奏能被提升到 120Hz**
   - `Info.plist` 声明高刷新是必需的，但还要确保 Flutter Engine 实际感知到 120Hz（某些版本存在节奏回退）。

2. **避免让 Flutter 作为“被动滚动”子视图**
   - 在原生滚动容器中嵌 Flutter，会放大节奏不一致的问题。
   - 更好的方式是把滚动交给 Flutter 或使用页面级的 add-to-app。

3. **观测真实帧率**
   - iOS 的 `Metrics` 和 `Core Animation` 工具可以看到 FlutterView 是否真的在 120Hz 刷新。
   - 不要只看 Flutter 的 `--profile`，它可能认为自己是 120Hz，但最终合成仍然被系统降频。

## 版本背景

- Flutter stable 3.38.5
- iOS 17 系列 ProMotion 设备
- add-to-app（FlutterViewController 嵌入）

## 总结

拖影的本质不是“Flutter 性能不够”，而是 **add-to-app 场景下渲染节奏不对齐**。

Info.plist 的 120Hz 声明只是让 Flutter 有资格参与更高频率的合成，而真正影响体验的是 **系统分配给 Flutter 的刷新预算**。

理解这个“时钟域不一致”的机制，才能在复杂场景下做正确的架构选择。
