---
title: Swift 路由指南：致 Flutter 开发者的导航迁移手册
date: 2026-02-03
tags:
  - Swift
  - iOS
  - Flutter
  - Navigation
  - 路由
---

## 引言

作为一个习惯了 Flutter `Navigator` 的开发者，转战 iOS 开发时，最容易混淆的概念就是 **Swift (UIKit)** 和 **SwiftUI** 的路由差异。

用户口中的 "Swift 开发" 通常指代两种完全不同的 UI 框架：
1.  **UIKit**: 传统的、基于 `UIViewController` 的命令式框架（类比 Flutter 的命令式 `Navigator 1.0`）。
2.  **SwiftUI**: 现代的、基于 `View` 的声明式框架（类比 Flutter 的 `Navigator 2.0` / `GoRouter`）。

本文将重点详解 **Swift (UIKit)** 的路由机制——这依然是目前国内大厂的主流选择，并对比 SwiftUI 的实现方式，帮助你建立完整的 iOS 导航知识体系。

## 核心概念映射表

| Flutter 概念 | Swift (UIKit) | SwiftUI |
| :--- | :--- | :--- |
| `Navigator` (栈管理者) | `UINavigationController` | `NavigationStack` |
| `Route` (页面对象) | `UIViewController` | `View` |
| `push()` | `pushViewController` | `NavigationLink` / `path.append()` |
| `pop()` | `popViewController` | `path.removeLast()` |
| `pushReplacement()` | 手动修改 `viewControllers` 数组 | 替换 `path` 数组内容 |
| `pushAndRemoveUntil()` | `setViewControllers` | 重置 `path` = [NewView] |
| `showDialog` / `Modal` | `present` (模态跳转) | `.sheet` / `.fullScreenCover` |

---

## 一、 Swift (UIKit)：手动挡的精密操控

在 UIKit 中，**导航即对象管理**。每一个页面都是一个 `UIViewController` 实例，而 `UINavigationController` 是管理这些实例的容器。

### 1. 基础结构：UINavigationController
这东西就是 Flutter 中 `Navigator` 的真身。它维护了一个 `viewControllers` 数组（栈）。

```swift
// 在 SceneDelegate 或 AppDelegate 中
let homeVC = HomeViewController()
// 初始化导航控制器，把 HomeVC 作为栈底
let nav = UINavigationController(rootViewController: homeVC)
window.rootViewController = nav 
```

### 2. 标准跳转 (Push & Pop)

**Push (Flutter: `Navigator.push`)**
```swift
let detailVC = DetailViewController()
detailVC.hidesBottomBarWhenPushed = true // 隐藏底部 TabBar（Flutter 默认行为）
// self 指的是当前的 ViewController
self.navigationController?.pushViewController(detailVC, animated: true)
```

**Pop (Flutter: `Navigator.pop`)**
```swift
self.navigationController?.popViewController(animated: true)
```

### 3. 高级栈操作 (Replace & RemoveUntil)
这是 Flutter 开发者最困惑的地方。UIKit 没有直接的 `pushReplacement` API，你需要**直接操作 Stack 数组**。

**场景 A: Push Replacement (进新页，删旧页)**
Flutter: `Navigator.pushReplacement(...)`
Swift:
```swift
guard let nav = self.navigationController else { return }

// 1. 获取当前栈的副本
var stack = nav.viewControllers
// 2. 移除最后一个（也就是当前页）
stack.removeLast()
// 3. 把新页面加进去
let newVC = TargetViewController()
stack.append(newVC)
// 4. 一次性赋值，UIKit 会智能处理动画
nav.setViewControllers(stack, animated: true)
```

**场景 B: Push And Remove Until (清空栈，只留新页)**
Flutter: `Navigator.pushAndRemoveUntil(...)`
Swift:
```swift
let newRoot = LoginViewController()
// 直接把栈覆盖为一个只有新页面的数组
self.navigationController?.setViewControllers([newRoot], animated: true)
```

**场景 C: Pop Until (回退到指定页)**
Flutter: `Navigator.popUntil(...)`
Swift:
```swift
let viewControllers = self.navigationController?.viewControllers ?? []
// 遍历栈找到目标 VC
if let targetVC = viewControllers.first(where: { $0 is HomeViewController }) {
    // 它是 popTo，不是 pop
    self.navigationController?.popToViewController(targetVC, animated: true)
}
```

### 4. 模态跳转 (Present)
这是和 Push 完全不同的维度。Push 是左右滑进（在栈内），Present 是从下往上盖上来（新开一个栈）。
Flutter 把 Dialog 和 Page 混在一起了，但 iOS 分得很开。

**Flutter:** `Navigator.push(..., fullscreenDialog: true)`
**Swift:**
```swift
let loginVC = LoginViewController()
// 如果需要导航栏，通常要再套一个 NavController
let navWrapper = UINavigationController(rootViewController: loginVC)
navWrapper.modalPresentationStyle = .fullScreen // 全屏还是卡片式
self.present(navWrapper, animated: true, completion: nil)
```

**关闭模态:**
```swift
self.dismiss(animated: true, completion: nil)
```

---

## 二、 SwiftUI：声明式的状态驱动

（这部分主要用于对比，展示未来的方向）

SwiftUI 的导航经历了从 `NavigationView` (已废弃) 到 **`NavigationStack`** (iOS 16+) 的重大变革。现在的 `NavigationStack` 与 Flutter 的 `GoRouter` 或 `Navigator 2.0` 非常相似：**路由状态驱动**。

### 1. 声明式定义 (NavigationStack)
你需要显式地定义一个 `NavigationStack` 包裹你的根视图。

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            HomeView()
        }
    }
}
```

### 2. 基础 Push (NavigationLink)
最简单的跳转，类似 HTML 的 `<a>` 标签。
```swift
NavigationLink("Go to Detail", destination: DetailView())
```

### 3. 可控路由 (Programmatic Navigation) - 重点！
这才是 Flutter 开发者想要的东西。我们需要一个 `path` 变量来控制路由栈。

```swift
struct ContentView: View {
    // 这里的 path 就像 Flutter 的路由栈状态
    @State private var path = [String]() // 假设存的是路由 ID

    var body: some View {
        NavigationStack(path: $path) {
            VStack {
                Button("Push Detail") {
                    path.append("detail_1") // 等同于 Navigator.push
                }
            }
            .navigationDestination(for: String.self) { routeId in
                // 类似于 Flutter 的 onGenerateRoute
                if routeId == "detail_1" {
                    DetailView()
                }
            }
        }
    }
}
```

### 4. 实现 Flutter 的高级操作

**场景 A: Pop (返回)**
Flutter: `Navigator.pop(context)`
SwiftUI: 
```swift
path.removeLast()
```

**场景 B: Pop to Root (返回首页)**
Flutter: `Navigator.popUntil(context, (route) => route.isFirst)`
SwiftUI:
```swift
path.removeAll() // 清空数组，自动回到根视图
```

**场景 C: Replace (替换当前页)**
Flutter: `Navigator.pushReplacement(...)`
SwiftUI:
```swift
if !path.isEmpty {
    path.removeLast()
}
path.append("newItem")
```

## 三、 传参机制对比

**Flutter:**
通常通过构造函数传参：
```dart
DetailView(user: currentUser)
```

**Swift (UIKit):**
同样是构造函数或属性赋值：
```swift
let vc = DetailViewController()
vc.user = currentUser
nav.push(vc)
```

**SwiftUI:**
利用 `navigationDestination` 解耦：
```swift
// 定义路由模型（Hashable 是必须的，类似 Flutter 的 Arguments）
struct UserRoute: Hashable {
    let id: Int
    let name: String
}

// 在 Stack 中
.navigationDestination(for: UserRoute.self) { user in
    UserProfileView(userId: user.id, userName: user.name)
}

// 跳转
path.append(UserRoute(id: 1, name: "Zhang"))
```

## 四、 模态弹窗 (Modal / Present)

在 iOS 中，Push 是水平进入（导航栈），Present 是垂直弹起（模态）。
Flutter 将两者混淆都在 `Navigator` 里（通过 `fullscreenDialog: true`）。但在 iOS 中它们是严格区分的。

**SwiftUI 弹窗 (Sheet):**
需要一个 `Bool` 状态绑定。

```swift
struct HomeView: View {
    @State private var showSheet = false

    var body: some View {
        Button("Show Modal") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            LoginView() // 这里的 View 拥有独立的上下文
        }
    }
}
```

## 三、 核心思维对比总结

| 维度 | Swift (UIKit) | SwiftUI | Flutter |
| :--- | :--- | :--- | :--- |
| **驱动模式** | **命令式 (Imperative)**<br>拿到控制器对象，发号施令。 | **声明式 (Declarative)**<br>改变 State 数组，UI 自动响应。 | 混合态<br>`Navigator 1.0` 像 UIKit<br>`GoRouter` 像 SwiftUI |
| **页面粒度** | **Heavy (ViewController)**<br>每个页面是很重的对象，有完整的生命周期。 | **Light (Struct View)**<br>页面只是纯数据描述，极轻量。 | **Medium (Widget)**<br>Widget 轻量，但 `Route` 对象较重。 |
| **灵活性** | **极高**<br>你可以随心所欲地修改 `viewControllers` 数组，甚至可以插入中间页。 | **高**<br>通过 `path` 数组操作，逻辑清晰但受限于框架 API。 | **高**<br>Flutter 的 Overlay 和 Route 机制非常灵活。 |

### 给 Flutter 专家的建议

1.  **如果你在写纯 Swift (UIKit) 项目**：
    不要试图寻找 `Navigator.pushNamed`。
    你要习惯 `let vc = MyVC(); nav.push(vc)` 这种“手工作坊”式的写法。
    对于复杂的跳转（如登录成功后重置首页），请熟练使用 `setViewControllers`。

2.  **关于 `pushReplacement` 的封装**：
    由于 UIKit 没提供这个 API，建议写一个 `UINavigationController` 的 Extension：

    ```swift
    extension UINavigationController {
        func pushReplacement(viewController: UIViewController, animated: Bool = true) {
            var currentStack = self.viewControllers
            if !currentStack.isEmpty {
                currentStack.removeLast()
            }
            currentStack.append(viewController)
            self.setViewControllers(currentStack, animated: animated)
        }
    }
    ```
    这样你就可以像 Flutter 一样写代码了：
    `navigationController?.pushReplacement(viewController: newVC)`

3.  **理解 Present vs Push**：
    Flutter 开发者最容易滥用 Push。在 iOS HIG 设计规范中，**流程内**的页面用 Push（如：列表->详情），**流程外**的任务用 Present（如：写文章、登录框）。这不仅是动画的区别，更是语义的区别。

