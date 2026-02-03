---
title: Dart 编译启示录：JIT、AOT 与“消失”的反射
date: 2026-02-03
tags:
  - Dart
  - Compiler
  - JIT
  - AOT
  - Reflection
---

## 引言

Dart 是一门性格“分裂”的语言。

在开发阶段，它像 JavaScript 一样灵动，修改代码后按下 `r` 键，应用瞬间更新（Hot Reload）；而在发布到生产环境时，它又像 C++ 一样严谨、高效，编译成机器码后性能强悍。

这种“开发体验”与“运行性能”兼得的背后，是 Dart 独特且强大的双重编译与执行模式：**JIT (Just-In-Time)** 和 **AOT (Ahead-Of-Time)**。

然而，代价也是显而易见的。很多从 Java 或 C# 转过来的开发者最不习惯的一点就是：**Dart 的生产环境里没有反射（Reflection）**。

这篇文章将深入底层，揭示这背后的技术取舍。

## JIT：为“快”而生的开发模式

在 `flutter run`（Debug 模式）时，Dart 运行在 JIT 模式下。

### 1. 增量编译与 Kernel Binary
当你按下 Hot Reload 时，Dart 并不是重新编译整个 App。
Dart VM 使用了一个中间格式叫 **Kernel Binary (.dill)**。前端编译器（Common Front End, CFE）只计算**文件修改的差异（Delta）**，然后将其通过 RPC 发送给手机上的 Dart VM。

### 2. 类比 JVM 的动态性
JIT 模式下的 Dart VM 包含了完整的编译器和分析器。它在运行时动态解析代码，甚至可以在运行时修改类的结构（添加方法、修改实现）。
这使得 `dart:mirrors`（反射库）在 JIT 模式下是**可用**的。因为 VM 知道所有的类信息、方法表和符号表，它就在内存里躺着。

但是，JIT 的缺点也很明显：
- **启动慢**：需要先解析代码再执行。
- **预热成本**：代码需要运行多次（Warmup）才能被优化编译器（Optimizing Compiler）生成高效的机器码。
- **体积大**：必须打包整个 Dart VM 和源码/字节码。

## AOT：为“稳”而生的生产模式

当你运行 `flutter build ipa/apk --release` 时，Dart 切换到了 AOT 模式。

### 1. 封闭世界假设 (Closed World Assumption)
这是 AOT 编译的核心哲学。
编译器假设：**“凡是编译期间我看不见被调用的代码，在运行时都绝对不会被用到。”**

基于这个假设，AOT 编译器（gen_snapshot）进行了一项至关重要的操作：**Tree Shaking（摇树优化）**。
它从 `main()` 函数开始，顺藤摸瓜，标记所有可达的函数和类。凡是没被标记的，统统丢弃，**连符号信息都不保留**。

### 2. 直接生成机器码
跳过字节码，直接生成目标平台的汇编指令（ARM64/x86）。且运行时不再包含编译器。
这带来了：
- **极速启动**：操作系统加载完指令直接执行。
- **稳定性能**：没有 JIT 的 de-optimization（去优化）抖动。
- **最小包体积**：没用的代码全没了。

## 为什么反射（`dart:mirrors`）必须死？

现在我们终于可以回答这个问题了：为什么生产环境不能用反射？

很多开发者认为：“为了性能，禁用了反射”。这话只对了一半。
核心矛盾在于：**反射机制与 Tree Shaking 是天敌。**

### 1. 动态调用的不可预测性
反射允许你写出这样的代码：

```dart
String methodName = "sayHello";
// 编译器在编译时，根本无法知道这个字符串对应哪个方法
mirrorSystem.invoke(methodName, []);
```

如果 AOT 编译器看到了这行代码，它陷入了逻辑死循环：
1. 为了支持这段代码，我必须保留**所有**类的**所有**方法，因为 `methodName` 可能是任何字符串。
2. 如果保留所有方法，Tree Shaking 就完全失效了。
3. 一个 Hello World 的 Flutter App，如果因为一行反射代码而引入了 Flutter framework 里上万个未使用的 Widget 和方法，包体积可能从 10MB 暴涨到 50MB 甚至更多。

### 2. 元数据膨胀
反射不仅需要代码存在，还需要大量的元数据（Metadata）：方法名、参数名、注解等。在 JIT 模式下，这些本身就在内存里；但在 AOT 模式下，如果要保留这些信息，二进制文件的大小会进一步失控。

### 3. 多平台约束
iOS 等平台（Store 审核政策）严格限制应用程序在运行时动态生成或执行新的机器码（Writable implies Executable, W^X）。虽然基本的反射调用不一定违反此规则，但为了支持完整反射而通常伴随的动态特性，很容易在大厂合规检测或系统安全策略上触雷。

## 解决方案：静态元编程 (Static Metaprogramming)

既然运行时反射（Runtime Reflection）行不通，Dart 社区给出的答案是：**编译时反射（Compile-time Reflection）**。

### 1. Code Generation (代码生成)
这就是为什么 Flutter/Dart 项目中 `build_runner` 如此泛滥的原因。
- **Json_serializable**
- **Retrofit**
- **Freezed**

它们做的事情本质上是：**在编译期间，明确地扫描注解，生成辅助代码。**
比如 `Person.g.dart` 里硬编码了 `map['name'] = instance.name`。
这让 Tree Shaking 依然有效，因为生成的代码是静态引用的，编译器看得见、摸得着。

### 2. Dart Macros (宏) - 探索与暂停

![macros_discontinued](https://pub.dev/packages/macros/score)

目前的 `build_runner` 最大的缺点是慢（文件 IO 开销）且体验割裂（生成一堆 `.g.dart`）。
Dart 团队曾长期研发 **Macros（宏）** 系统，旨在允许在编译器的前端阶段直接操作 AST（抽象语法树），就地生成代码。

**遗憾的是，该提案目前已处于暂停/废弃状态（Discontinued）。**
正如 Pub.dev 上该库已停止更新（上一次更新还是 2 年前），由于实现复杂度过高以及对编译器性能的潜在影响，Dart 团队重新评估了这一路线。这意味着在未来很长一段时间内，我们可能仍需依赖 `build_runner` 体系，或者等待基于 **Augmentations** 等更轻量级特性的新方案出现。

## 结语

Dart 禁用生产环境反射，并非技术上的无能，而是对 **移动端工程化（体积、启动速度、内存）** 极致追求下的主动选择。

理解了 **Closed World Assumption**，你就能理解 Dart 编译器的一切行为逻辑。在 Flutter 的世界里，我们牺牲了运行时的灵活性，换取了极致的性能与确定性。
