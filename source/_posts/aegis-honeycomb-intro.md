---
title: 介绍一下我自己写的状态管理库 aegis_honeycomb
date: 2026-03-04
tags:
  - Flutter
  - State Management
  - Dart
  - aegis_honeycomb
---

## 🍯 为什么我要写一个新的状态管理库？

在 Flutter 社区，状态管理方案百花齐放。从早期的 Provider，到后来的 Riverpod、BLoC 和 GetX，每一个库都有其独特的优势。然而，在实际的大型项目开发中，我发现始终有一些痛点难以被优雅地解决：

1. **模板代码与代码生成（Codegen）**：许多库（如 Riverpod 2.0+）强依赖 `build_runner` 代码生成，这在大型项目中会导致编译速度缓慢，让开发体验大打折扣。
2. **状态（State）与副作用（Effect）的混淆**：有时候我们只需要触发一个单纯的事件（如：页面跳转、Toast 弹窗），而在响应式状态管理中，处理这种“一次性事件”往往比较别扭。
3. **脱离 BuildContext 的依赖注入**：在纯 Dart 业务逻辑（如 Service、Repository）中访问和修改状态时，往往需要费尽周折地传递 `ref` 或 `context`。

为了解决这些问题，我开发了 **[aegis_honeycomb](https://pub.dev/packages/aegis_honeycomb)**。它是一款精简、类型安全、**无代码生成 (Codegen-Free)** 的 Flutter 状态管理库。

## ✨ 核心特性

`aegis_honeycomb` 的设计哲学是在保持最小心智负担的同时，提供强大的工程化能力。

* 🎯 **无上下文依赖 (Context-Free)**：可以直接在纯 Dart 业务逻辑中，通过全局容器访问状态。
* ⚡ **自动依赖追踪**：使用 `Computed` 时会自动从 `watch` 中追踪依赖，无需手动声明。
* 📡 **分离 State 与 Effect**：清晰区分可重播的状态与一次性的事件。
* 🎭 **作用域与局部重写 (Scope/Override)**：提供灵活的依赖注入和强大的 Mock 测试能力。
* 🔄 **无代码生成 (No Codegen)**：纯 Dart 实现，告别漫长的 `build_runner` 等待。
* 🔒 **类型安全**：借助 Dart 强大的泛型系统保证类型推导。

---

## 🚀 快速上手

你可以通过简单的三个步骤在项目中引入 `aegis_honeycomb`。

### 1. 定义状态

我们支持读写状态、衍生状态、异步状态和一次性事件。

```dart
import 'package:aegis_honeycomb/honeycomb.dart';

// 基础读写状态
final counterState = StateRef(0);

// 衍生状态 (自动追踪依赖)
final doubledCounter = Computed((watch) => watch(counterState) * 2);

// 异步状态
final userProfile = Computed.async((watch) async {
  final userId = watch(currentUserId);
  return await api.fetchUser(userId);
});

// 一次性事件 (如 Toast)
final toastEffect = Effect<String>();
```

### 2. 提供全栈容器

你可以将容器放在全局，也可以像 Riverpod 一样包裹在顶层。

```dart
final appContainer = HoneycombContainer();

void main() {
  runApp(
    HoneycombScope(
      container: appContainer,
      child: MyApp(),
    ),
  );
}
```

### 3. 在 UI 中使用

使用 `HoneycombConsumer` 监听并构建 UI。

```dart
class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return HoneycombConsumer(
      builder: (context, ref, child) {
        // 自动完成监听，当 count 或 doubled 改变时刷新
        final count = ref.watch(counterState);
        final doubled = ref.watch(doubledCounter);

        return Column(
          children: [
            Text('Count: $count'),
            Text('Doubled: $doubled'),
            ElevatedButton(
              onPressed: () {
                final container = HoneycombScope.readOf(context);
                container.write(counterState, count + 1);
              },
              child: Text('Increment'),
            ),
          ],
        );
      },
    );
  }
}
```

---

## 🎯 亮点功能解析

### 1. 明确区分 State 与 Effect

在传统的响应式流中，如果你监听了一个 error string 并在页面显示 Toast。当页面重建时，这个历史遗留的 error string 可能会导致 Toast 重复弹出。

在 Honeycomb 中，我们引入了 `Effect`。

```dart
// State：可重播、可保留，始终返回最新值
final userName = StateRef('Guest');

// Effect：一次性事件，用完即弃，绝不存储历史
final showToast = Effect<String>(strategy: EffectStrategy.drop);
```

### 2. 在纯 Dart 业务逻辑中使用 (脱离 Context)

我们在写 Repository 或 Service 时，经常苦恼如何和 UI 层共享数据源。通过全局 `HoneycombContainer`，你可以在任何地方读写状态和分发事件。

```dart
// app_globals.dart
final appContainer = HoneycombContainer();

class AuthService {
  void logout() {
    // 读取状态
    final currentUser = appContainer.read(userState);
    
    // 修改状态
    appContainer.write(userState, null);
    
    // 发送路由或 UI 事件
    appContainer.emit(navigationEffect, '/login');
  }
}
```

### 3. 灵活的作用域 (Scope) 与自动化测试

局部覆盖 (Override) 不论是对多主题切换还是对单元测试 Mock 都是绝对的杀手锏。当遇到 `override` 时，Honeycomb 不会向上层容器查找，而是使用当前的覆盖逻辑。

```dart
HoneycombScope(
  overrides: [
    // 强制此子树中的主题为深色模式
    themeState.overrideWith(ThemeData.dark()),

    // 单元测试中替换为 Mock 数据
    userProfile.overrideWith(AsyncValue.data(MockUser())),
  ],
  child: DarkModePage(),
)
```

编写测试变得无比简单：

```dart
test('counter increments', () {
  final container = HoneycombContainer();
  
  expect(container.read(counterState), 0);
  
  container.write(counterState, 1);
  
  expect(container.read(counterState), 1);
  expect(container.read(doubledCounter), 2); // 自动计算衍生结果
});
```

## 结语

`aegis_honeycomb` 是我结合过往在 Flutter 大中型项目中遇到的无数业务痛点设计出的轮子。它在保留主流响应式框架依赖追踪、自动派发优点的同时，大胆舍弃了繁重的代码生成，并清晰划分了状态与副作用的边界。

欢迎大家去 pub 上试用并点个 👍！
传送门：[pub.dev/packages/aegis_honeycomb](https://pub.dev/packages/aegis_honeycomb) 
开源仓库：[GitHub](https://github.com/AegisLabsOrg/Honeycomb) 

如果有任何建议或 Issue，欢迎在 GitHub 上交流。
