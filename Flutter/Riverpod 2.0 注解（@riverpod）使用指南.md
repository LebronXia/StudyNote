

# Riverpod 2.0 注解（@riverpod）使用指南

## 1. 添加依赖

在 `pubspec.yaml` 中添加依赖：
```yaml
dependencies:
  flutter_riverpod: ^2.0.0
  riverpod_annotation: ^2.0.0

dev_dependencies:
  build_runner: ^2.0.0
  riverpod_generator: ^2.0.0
```

运行安装命令：

bash

```bash
flutter pub get
```

------

## 2. 初始化应用根作用域

在 `main.dart` 中包裹 `ProviderScope`：

dart

```dart
void main() {
  runApp(
    ProviderScope( // 必须包裹整个应用
      child: MyApp(),
    ),
  );
}
```

------

## 3. 创建 Provider

### 同步 Provider

dart

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'counter_provider.g.dart';

@riverpod
int counter(CounterRef ref) { 
  return 0;
}
```

### 异步 Provider

dart

```dart
@riverpod
Future<String> fetchUser(FetchUserRef ref, int userId) async {
  return await http.get('https://api.example.com/users/$userId');
}
```

### 状态管理（StateNotifier）

dart

```dart
@riverpod
class CounterNotifier extends _$CounterNotifier {
  @override
  int build() => 0; // 初始状态

  void increment() => state++;
}
```

------

## 4. 生成代码

运行生成命令：

bash

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

------

## 5. 在 UI 中使用

### 访问同步 Provider

dart

```dart
class CounterWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('Count: $count');
  }
}
```

### 访问异步 Provider

dart

```dart
userAsync.when(
  loading: () => CircularProgressIndicator(),
  error: (e, _) => Text('Error: $e'),
  data: (user) => Text(user),
);
```

### 访问 StateNotifier

dart

```dart
ElevatedButton(
  onPressed: () => ref.read(counterNotifierProvider.notifier).increment(),
  child: Text('Count: $count'),
);
```

------

## 6. 高级用法

### 依赖注入

dart

```dart
@riverpod
Future<User> userDetails(UserDetailsRef ref) async {
  final userId = ref.watch(currentUserIdProvider);
  return await fetchUser(userId);
}
```

### 自动释放资源

dart

```dart
@Riverpod(keepAlive: false)
int tempCounter(TempCounterRef ref) {
  return 0;
}
```

### 监听状态变化

dart

```dart
ref.listen<int>(counterProvider, (previous, next) {
  print('Counter changed from $previous to $next');
});
```

------

## 7. 常见问题

### Provider not found

- 检查 `ProviderScope` 是否包裹根 Widget
- 确认生成了 `.g.dart` 文件

### 参数化 Provider

dart

```dart
@riverpod
Future<User> fetchUser(FetchUserRef ref, {required int userId}) async {
  return await getUser(userId);
}

// 使用
ref.watch(fetchUserProvider(userId: 123));
```