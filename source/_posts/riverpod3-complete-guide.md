---
title: Riverpod 3.0 完全指南
date: 2026-01-06 10:00:00
tags:
  - Flutter
  - Riverpod
  - 状态管理
categories:
  - 编程
  - Flutter
---

# Riverpod 3.0 完全指南

## 简介

Riverpod 是一个强大的 Flutter 状态管理框架，它是 Provider 的完全重写版本。Riverpod 3.0 带来了许多重大改进和新特性，本文将全面介绍 Riverpod 3.0 的核心概念、使用方法和最佳实践。

### 什么是 Riverpod？

Riverpod 是一个**响应式缓存和数据绑定框架**，它提供了：

- ✅ 声明式编程
- ✅ 原生网络请求支持
- ✅ 自动加载/错误处理
- ✅ 编译时安全
- ✅ 类型安全的查询参数
- ✅ 易于测试
- ✅ 支持纯 Dart（服务器/CLI等）
- ✅ 状态易于组合
- ✅ 内置下拉刷新支持
- ✅ 自定义 lint 规则
- ✅ 内置重构工具
- ✅ 热重载支持

<!-- more -->

## 安装和配置

### 1. 安装依赖

根据你的需求选择合适的包：

```bash
# Flutter 项目（基础版本）
flutter pub add flutter_riverpod

# 使用代码生成（推荐）
flutter pub add flutter_riverpod riverpod_annotation dev:riverpod_generator dev:build_runner dev:custom_lint dev:riverpod_lint
```

### 2. 配置 analysis_options.yaml

启用 riverpod_lint 以获得更好的开发体验：

```yaml
analyzer:
  plugins:
    - riverpod_lint

plugins:
  riverpod_lint:
    # 最新版本见 https://pub.dev/packages/riverpod_lint
```

### 3. Hello World 示例

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// 创建一个 Provider
final helloWorldProvider = Provider((ref) => 'Hello world');

void main() {
  runApp(
    // 使用 ProviderScope 包裹整个应用
    ProviderScope(
      child: MyApp(),
    ),
  );
}

// 使用 ConsumerWidget 而不是 StatelessWidget
class MyApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final String value = ref.watch(helloWorldProvider);

    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: const Text('Example')),
        body: Center(
          child: Text(value),
        ),
      ),
    );
  }
}
```

## Riverpod 3.0 核心概念

### 1. Provider

Provider 是 Riverpod 的核心，它是一个声明式的状态容器。

#### Provider 类型

**基础 Provider**
```dart
final configProvider = Provider<Configuration>((ref) {
  return Configuration();
});
```

**FutureProvider - 异步数据**
```dart
@riverpod
Future<String> boredSuggestion(Ref ref) async {
  final response = await http.get(
    Uri.https('boredapi.com', '/api/activity'),
  );
  final json = jsonDecode(response.body) as Map;
  return json['activity']! as String;
}
```

**StreamProvider - 流数据**
```dart
@riverpod
Stream<int> counter(Ref ref) async* {
  int count = 0;
  while (true) {
    await Future.delayed(Duration(seconds: 1));
    yield count++;
  }
}
```

**NotifierProvider - 可变状态**
```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
  void decrement() => state--;
}
```

**AsyncNotifierProvider - 异步可变状态**
```dart
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    return await fetchTodos();
  }

  Future<void> addTodo(Todo todo) async {
    state = AsyncLoading();
    state = await AsyncValue.guard(() async {
      await api.addTodo(todo);
      return [...await future, todo];
    });
  }
}
```

### 2. Ref - Provider 的引用

Ref 对象是与 Provider 交互的核心工具。

#### ref.watch - 监听状态变化

```dart
@riverpod
Widget example(Ref ref) {
  // 当 counterProvider 变化时，会重新构建
  final count = ref.watch(counterProvider);
  return Text('$count');
}
```

#### ref.read - 一次性读取

```dart
void onPressed() {
  // 仅读取一次，不监听变化
  final count = ref.read(counterProvider);
  print(count);
}
```

#### ref.listen - 监听并执行副作用

```dart
ref.listen(counterProvider, (previous, next) {
  // 当值变化时执行
  print('Counter changed from $previous to $next');
});
```

#### ref.mounted - 检查是否仍然挂载

```dart
Future<void> addTodo(String title) async {
  final newTodo = await api.addTodo(title);
  
  // 检查 provider 是否还在挂载状态
  if (!ref.mounted) return;
  
  state = [...state, newTodo];
}
```

### 3. 读取 Provider 的方式

#### 在 Widget 中读取

**方式 1: ConsumerWidget**
```dart
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final value = ref.watch(myProvider);
    return Text('$value');
  }
}
```

**方式 2: Consumer**
```dart
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Consumer(
      builder: (context, ref, child) {
        final value = ref.watch(myProvider);
        return Text('$value');
      },
    );
  }
}
```

**方式 3: ConsumerStatefulWidget**
```dart
class MyWidget extends ConsumerStatefulWidget {
  @override
  ConsumerState<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends ConsumerState<MyWidget> {
  @override
  Widget build(BuildContext context) {
    final value = ref.watch(myProvider);
    return Text('$value');
  }
}
```

## Riverpod 3.0 新特性

### 1. 离线持久化（实验性）

Provider 现在可以选择性地持久化到数据库中。

```dart
final storageProvider = FutureProvider<JsonSqFliteStorage>((ref) async {
  return JsonSqFliteStorage.open(
    join(await getDatabasesPath(), 'riverpod.db'),
  );
});

class Todo {
  const Todo({
    required this.id,
    required this.description,
    required this.completed,
  });

  Todo.fromJson(Map<String, dynamic> json)
      : id = json['id'] as int,
        description = json['description'] as String,
        completed = json['completed'] as bool;

  final int id;
  final String description;
  final bool completed;

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'description': description,
      'completed': completed,
    };
  }
}

@riverpod
class Todos extends _$Todos {
  @override
  Future<List<Todo>> build() async {
    // 调用 persist 在 build 方法开始时
    persist(
      ref.watch(storageProvider.future),
      key: 'todos',
      encode: jsonEncode,
      decode: (json) {
        final decoded = jsonDecode(json) as List;
        return decoded.map((e) => Todo.fromJson(e as Map<String, Object?>)).toList();
      },
    );

    // 从服务器获取数据
    final todos = await fetchTodos();
    return todos;
  }

  Future<void> add(Todo todo) async {
    state = AsyncData([...await future, todo]);
  }
}
```

### 2. Mutations（实验性）

Mutations 解决了两个问题：
1. 让 UI 能够响应副作用（如表单提交、按钮点击）
2. 解决了 `onPressed` 回调与自动销毁的问题

```dart
final addTodoMutation = Mutation<void>();

class AddTodoButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final addTodo = ref.watch(addTodoMutation);

    return switch (addTodo) {
      MutationIdle() => ElevatedButton(
        onPressed: () {
          addTodoMutation.run(ref, (tsx) async {
            await tsx.get(todoListProvider.notifier).addTodo('New Todo');
          });
        },
        child: const Text('Submit'),
      ),
      MutationPending() => const CircularProgressIndicator(),
      MutationError() => ElevatedButton(
        onPressed: () { /* 重试逻辑 */ },
        child: const Text('Retry'),
      ),
      MutationSuccess() => const Text('Todo added!'),
    };
  }
}
```

### 3. 自动重试

Provider 失败时会自动重试，使用指数退避策略。

```dart
void main() {
  runApp(
    ProviderScope(
      retry: (retryCount, error) {
        // 跳过特定错误
        if (error is SomeSpecificError) return null;
        // 限制重试次数
        if (retryCount > 5) return null;
        // 自定义延迟
        return Duration(seconds: retryCount * 2);
      },
      child: MyApp(),
    ),
  );
}

// 或者针对单个 provider
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    // ...
  }
}

final todoListProvider = AsyncNotifierProvider<TodoList, List<Todo>>(
  TodoList.new,
  retry: (retryCount, error) {
    if (error is SomeSpecificError) return null;
    if (retryCount > 5) return null;
    return Duration(seconds: retryCount * 2);
  },
);
```

### 4. 泛型支持（代码生成）

```dart
@riverpod
T multiply<T extends num>(T a, T b) {
  return a * b;
}

// 使用
int integer = ref.watch(multiplyProvider<int>(2, 3));
double decimal = ref.watch(multiplyProvider<double>(2.5, 3.5));
```

### 5. 暂停/恢复支持

```dart
final subscription = ref.listen(
  todoListProvider,
  (previous, next) {
    // 处理新值
  },
);

subscription.pause();
subscription.resume();
```

### 6. API 统一

#### AutoDispose 接口移除

不再需要 `AutoDisposeNotifier`，统一使用 `Notifier`：

```dart
// 之前
class MyNotifier extends AutoDisposeNotifier<int> {
  @override
  int build() => 0;
}

// 现在
class MyNotifier extends Notifier<int> {
  @override
  int build() => 0;
}
```

#### FamilyNotifier 和 Notifier 合并

```dart
// 之前
class CounterNotifier extends FamilyNotifier<int, Argument> {
  @override
  int build(Argument arg) => 0;
}

// 现在
class CounterNotifier extends Notifier<int> {
  CounterNotifier(this.arg);
  final Argument arg;
  
  @override
  int build() => 0;
}
```

#### 统一的 Ref

不再有 `FutureProviderRef`、`StreamProviderRef` 等，只有一个 `Ref`：

```dart
// 之前
Example example(ExampleRef ref) {
  return Example();
}

// 现在
Example example(Ref ref) {
  return Example();
}
```

## AsyncValue 处理异步状态

AsyncValue 是处理异步数据的强大工具。

### 基本用法

```dart
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncValue = ref.watch(futureProvider);

    return switch (asyncValue) {
      AsyncData(:final value) => Text('Data: $value'),
      AsyncError(:final error) => Text('Error: $error'),
      AsyncLoading() => CircularProgressIndicator(),
    };
  }
}
```

### AsyncValue 方法

```dart
// when - 处理所有状态
asyncValue.when(
  data: (value) => Text('Data: $value'),
  loading: () => CircularProgressIndicator(),
  error: (error, stack) => Text('Error: $error'),
);

// whenOrNull - 可选处理
asyncValue.whenOrNull(
  data: (value) => Text('Data: $value'),
) ?? Text('Loading or Error');

// maybeWhen - 带默认值
asyncValue.maybeWhen(
  data: (value) => Text('Data: $value'),
  orElse: () => Text('Loading or Error'),
);
```

### AsyncValue 3.0 新特性

```dart
// value - 获取值（之前是 valueOrNull）
final value = asyncValue.value; // 可能为 null

// isFromCache - 是否来自缓存
if (asyncValue.isFromCache) {
  print('Data from cache');
}

// progress - 进度（仅在 AsyncLoading 时可用）
class MyNotifier extends AsyncNotifier<User> {
  @override
  Future<User> build() async {
    state = AsyncLoading(progress: 0.0);
    await fetchSomething();
    state = AsyncLoading(progress: 0.5);
    return User();
  }
}
```

## 组合 Providers

Riverpod 的强大之处在于能够轻松组合多个 providers。

### 基本组合

```dart
@riverpod
List<Todo> filteredTodos(Ref ref) {
  final todos = ref.watch(todosProvider);
  final filter = ref.watch(filterProvider);

  switch (filter) {
    case Filter.all:
      return todos;
    case Filter.completed:
      return todos.where((todo) => todo.completed).toList();
    case Filter.uncompleted:
      return todos.where((todo) => !todo.completed).toList();
  }
}
```

### Family 修饰符 - 参数化 Provider

```dart
@riverpod
Future<Todo> todo(Ref ref, int id) async {
  return await fetchTodo(id);
}

// 使用
final todo1 = ref.watch(todoProvider(1));
final todo2 = ref.watch(todoProvider(2));
```

### AutoDispose 修饰符 - 自动清理

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() {
    // 当不再使用时自动销毁
    ref.onDispose(() {
      print('Counter disposed');
    });
    return 0;
  }

  void increment() => state++;
}

// 使用 keepAlive 防止自动销毁
@riverpod
Future<Data> data(Ref ref) async {
  final data = await fetchData();
  ref.keepAlive(); // 永久保持
  return data;
}

// 或者使用定时器
@riverpod
Future<Data> data(Ref ref) async {
  final data = await fetchData();
  final link = ref.keepAlive();
  
  // 10秒后销毁
  Timer(Duration(seconds: 10), link.close);
  
  return data;
}
```

## 最佳实践

### 1. 使用代码生成

代码生成提供了更好的类型安全和开发体验：

```dart
// 推荐：使用代码生成
@riverpod
Future<User> user(Ref ref, String userId) async {
  return await fetchUser(userId);
}

// 不推荐：手动创建
final userProvider = FutureProvider.family<User, String>((ref, userId) async {
  return await fetchUser(userId);
});
```

### 2. Provider 命名规范

```dart
// 推荐的命名
@riverpod
Future<User> currentUser(Ref ref) async { ... }

@riverpod
class TodoList extends _$TodoList { ... }

// 生成的 provider 名称：
// - currentUserProvider
// - todoListProvider
```

### 3. 状态管理模式

```dart
// Model 层
class Todo {
  final String id;
  final String title;
  final bool completed;

  Todo({
    required this.id,
    required this.title,
    this.completed = false,
  });

  Todo copyWith({String? title, bool? completed}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      completed: completed ?? this.completed,
    );
  }
}

// Repository 层
@riverpod
class TodoRepository extends _$TodoRepository {
  @override
  FutureOr<void> build() {}

  Future<List<Todo>> fetchTodos() async {
    // API 调用
  }

  Future<void> addTodo(Todo todo) async {
    // API 调用
  }
}

// State 层
@riverpod
class TodoList extends _$TodoList {
  @override
  Future<List<Todo>> build() async {
    final repository = ref.read(todoRepositoryProvider.notifier);
    return await repository.fetchTodos();
  }

  Future<void> addTodo(String title) async {
    final todo = Todo(id: DateTime.now().toString(), title: title);
    
    // 乐观更新
    final previousState = state;
    state = AsyncData([...await future, todo]);

    try {
      final repository = ref.read(todoRepositoryProvider.notifier);
      await repository.addTodo(todo);
    } catch (e) {
      // 回滚
      state = previousState;
      rethrow;
    }
  }

  Future<void> toggleTodo(String id) async {
    state = AsyncData(
      (await future).map((todo) {
        if (todo.id == id) {
          return todo.copyWith(completed: !todo.completed);
        }
        return todo;
      }).toList(),
    );
  }
}
```

### 4. 错误处理

```dart
@riverpod
Future<User> user(Ref ref) async {
  try {
    return await fetchUser();
  } on NetworkException catch (e) {
    // 处理网络错误
    throw UserFriendlyException('Network error: ${e.message}');
  } catch (e) {
    // 处理其他错误
    throw UserFriendlyException('Unknown error');
  }
}

// 在 UI 中处理
class UserView extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider);

    return userAsync.when(
      data: (user) => UserCard(user: user),
      loading: () => CircularProgressIndicator(),
      error: (error, stack) {
        if (error is UserFriendlyException) {
          return ErrorCard(message: error.message);
        }
        return ErrorCard(message: 'Something went wrong');
      },
    );
  }
}
```

## 测试

Riverpod 3.0 提供了更好的测试工具。

### 1. ProviderContainer.test

```dart
void main() {
  test('counter increments', () {
    final container = ProviderContainer.test();
    final counter = container.read(counterProvider.notifier);

    expect(container.read(counterProvider), 0);
    
    counter.increment();
    expect(container.read(counterProvider), 1);
    
    // container 会自动清理
  });
}
```

### 2. 覆盖 Providers

```dart
test('with mocked data', () {
  final container = ProviderContainer.test(
    overrides: [
      userProvider.overrideWith((ref) async {
        return User(id: '1', name: 'Test User');
      }),
    ],
  );

  final user = container.read(userProvider);
  expect(user, isA<AsyncData>());
});
```

### 3. NotifierProvider.overrideWithBuild

```dart
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
}

test('override only build', () {
  final container = ProviderContainer.test(
    overrides: [
      counterProvider.overrideWithBuild((ref) {
        // 只模拟 build，increment 方法保持不变
        return 42;
      }),
    ],
  );

  expect(container.read(counterProvider), 42);
  
  container.read(counterProvider.notifier).increment();
  expect(container.read(counterProvider), 43);
});
```

### 4. Widget 测试

```dart
testWidgets('counter widget test', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      child: MaterialApp(
        home: CounterWidget(),
      ),
    ),
  );

  // 访问 ProviderContainer
  final container = tester.container();

  expect(find.text('0'), findsOneWidget);
  
  await tester.tap(find.byIcon(Icons.add));
  await tester.pump();
  
  expect(find.text('1'), findsOneWidget);
});
```

## 高级技巧

### 1. 生命周期管理

```dart
@riverpod
class MyNotifier extends _$MyNotifier {
  @override
  int build() {
    // Provider 创建时
    print('Provider created');

    // 监听器被添加时
    ref.onCancel(() {
      print('All listeners removed');
    });

    // 监听器恢复时
    ref.onResume(() {
      print('Listener added again');
    });

    // Provider 销毁时
    ref.onDispose(() {
      print('Provider disposed');
    });

    return 0;
  }
}
```

### 2. 依赖注入

```dart
// API 服务
@riverpod
ApiService apiService(Ref ref) {
  return ApiService(baseUrl: 'https://api.example.com');
}

// 使用 API 服务的 Repository
@riverpod
class UserRepository extends _$UserRepository {
  @override
  FutureOr<void> build() {}

  Future<User> fetchUser(String id) async {
    final api = ref.read(apiServiceProvider);
    return await api.getUser(id);
  }
}
```

### 3. 选择性监听

```dart
// select - 只在特定字段变化时重建
class UserView extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 只在 name 变化时重建
    final name = ref.watch(userProvider.select((user) => user.name));
    return Text(name);
  }
}
```

### 4. 弱监听

```dart
@riverpod
int example(Ref ref) {
  // 使用 weak 监听不会阻止自动销毁
  ref.listen(
    anotherProvider,
    weak: true,
    (previous, next) {},
  );

  return 0;
}
```

## 迁移指南

### 从 Provider 迁移

如果你正在使用 Provider 包，迁移到 Riverpod 的主要变化：

1. **不再需要 BuildContext**
   ```dart
   // Provider
   final value = Provider.of<T>(context);
   
   // Riverpod
   final value = ref.watch(provider);
   ```

2. **ProviderScope 替代 MultiProvider**
   ```dart
   // Provider
   MultiProvider(
     providers: [
       Provider(create: (_) => Something()),
     ],
     child: MyApp(),
   )
   
   // Riverpod
   ProviderScope(
     child: MyApp(),
   )
   ```

3. **ConsumerWidget 替代 Consumer**
   ```dart
   // Provider
   Consumer<T>(
     builder: (context, value, child) => ...
   )
   
   // Riverpod
   ConsumerWidget {
     Widget build(BuildContext context, WidgetRef ref) {
       final value = ref.watch(provider);
       ...
     }
   }
   ```

### 从 Riverpod 2.0 迁移到 3.0

主要变化：

1. **valueOrNull → value**
   ```dart
   // 2.0
   final value = asyncValue.valueOrNull;
   
   // 3.0
   final value = asyncValue.value;
   ```

2. **统一接口**
   ```dart
   // 2.0
   class MyNotifier extends AutoDisposeNotifier<int> { ... }
   
   // 3.0
   class MyNotifier extends Notifier<int> { ... }
   ```

3. **异常处理**
   ```dart
   // 3.0 中异常被包装在 ProviderException 中
   try {
     ref.watch(provider);
   } on ProviderException catch (e) {
     // 检查原始异常
     if (e.exception is MyException) { ... }
   }
   ```

## 性能优化

### 1. 使用 select 减少重建

```dart
// 不好：整个 User 对象变化都会重建
final user = ref.watch(userProvider);

// 好：只在 name 变化时重建
final name = ref.watch(userProvider.select((user) => user.name));
```

### 2. 避免在 build 中使用 ref.read

```dart
// 不好
Widget build(BuildContext context, WidgetRef ref) {
  final value = ref.read(provider); // 不会响应变化
  return Text('$value');
}

// 好
Widget build(BuildContext context, WidgetRef ref) {
  final value = ref.watch(provider); // 会响应变化
  return Text('$value');
}
```

### 3. 合理使用 autoDispose

```dart
// 对于一次性数据，使用 autoDispose
@riverpod
Future<User> userDetail(Ref ref, String userId) async {
  return await fetchUser(userId);
}

// 对于全局状态，不使用 autoDispose
@Riverpod(keepAlive: true)
Future<Config> appConfig(Ref ref) async {
  return await loadConfig();
}
```

## 调试技巧

### 1. 使用 ProviderObserver

```dart
class MyObserver extends ProviderObserver {
  @override
  void didUpdateProvider(
    ProviderBase provider,
    Object? previousValue,
    Object? newValue,
    ProviderContainer container,
  ) {
    print('Provider ${provider.name ?? provider.runtimeType} updated');
    print('Previous: $previousValue');
    print('New: $newValue');
  }

  @override
  void providerDidFail(
    ProviderObserverContext context,
    Object error,
    StackTrace stackTrace,
  ) {
    print('Provider failed: $error');
  }
}

void main() {
  runApp(
    ProviderScope(
      observers: [MyObserver()],
      child: MyApp(),
    ),
  );
}
```

### 2. 使用 Flutter DevTools

Riverpod 状态在 Flutter DevTools 中可见，可以实时查看和检查 provider 状态。

### 3. 命名 Providers

```dart
@riverpod
Future<User> user(Ref ref) async { ... }

// 生成的 provider 会有名称：userProvider
```

## 总结

Riverpod 3.0 是一个强大且成熟的状态管理解决方案，它提供了：

1. **类型安全**：编译时捕获错误
2. **易于测试**：Provider 可以轻松覆盖和测试
3. **可组合**：Provider 可以轻松组合
4. **性能优化**：自动优化重建和内存使用
5. **开发体验**：强大的 lint 规则和重构工具
6. **现代特性**：离线持久化、mutations、自动重试等

### 推荐的学习路径

1. 从基础的 Provider 和 ConsumerWidget 开始
2. 学习使用代码生成（@riverpod 注解）
3. 理解异步数据处理（AsyncValue）
4. 掌握 Provider 组合
5. 学习高级特性（mutations、persistence）
6. 实践最佳实践和性能优化

### 参考资源

- [官方文档](https://riverpod.dev/)
- [GitHub 仓库](https://github.com/rrousselgit/riverpod)
- [Discord 社区](https://discord.gg/GSt793j6eT)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/riverpod)

通过掌握 Riverpod 3.0，你将能够构建更加健壮、可维护和高性能的 Flutter 应用！
