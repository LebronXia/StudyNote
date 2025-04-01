
## 一、合理选择 Provider 类型

### 1. FutureProvider
**适用场景**：单个异步接口请求  
```dart
final userProvider = FutureProvider<User>((ref) async {
  return await fetchUser();
});

// 参数化用法
final userDetailProvider = FutureProvider.family<User, String>((ref, userId) {
  return fetchUserById(userId);
});
```

### 2. StateNotifierProvider

**适用场景**：复杂状态管理（分页/表单）

dart

```dart
final paginationProvider = StateNotifierProvider<PaginationNotifier, List<Item>>((ref) {
  return PaginationNotifier(ref.read(apiClientProvider));
});
```

### 3. StateProvider

**适用场景**：简单状态管理

dart

```dart
final filterProvider = StateProvider<FilterType>((ref) => FilterType.all);
```

------

## 二、分层管理状态

### 1. 基础数据层

dart

```dart
final postsProvider = FutureProvider<List<Post>>((ref) => fetchPosts());
```

### 2. 派生状态层

dart

```dart
final filteredPostsProvider = Provider<List<Post>>((ref) {
  final posts = ref.watch(postsProvider);
  final filter = ref.watch(filterProvider);
  return posts.where((post) => post.type == filter).toList();
});
```

------

## 三、生命周期优化

### 1. 自动销毁

dart

```dart
final tempDataProvider = StateProvider.autoDispose<int>((ref) => 0);
```

### 2. 跨页面共享

dart

```dart
// 根作用域直接声明
final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) => AuthNotifier());
```

------

## 四、代码架构设计

### 1. Repository 模式

dart

```dart
final userRepositoryProvider = Provider<UserRepository>((ref) {
  return UserRepository(ref.read(apiClientProvider));
});

final userProvider = FutureProvider<User>((ref) async {
  return ref.watch(userRepositoryProvider).fetchUser();
});
```

### 2. MVVM 架构

dart

```dart
class UserViewModel extends StateNotifier<BaseState<User>> {
  UserViewModel(this.ref) : super(const Initial());
  final Ref ref;

  Future<void> loadUser() async {
    state = const Loading();
    try {
      final user = await ref.read(userRepositoryProvider).fetchUser();
      state = Success(user);
    } catch (e) {
      state = Error(e.toString());
    }
  }
}

final userViewModelProvider = StateNotifierProvider<UserViewModel, BaseState<User>>((ref) {
  return UserViewModel(ref);
});
```

------

## 五、界面层集成

### 1. 多状态监听

dart

```dart
class UserProfilePage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userState = ref.watch(userViewModelProvider);
    final postsState = ref.watch(postsProvider);

    return userState.when(
      loading: () => LoadingIndicator(),
      error: (msg) => ErrorView(msg),
      success: (user) => Scaffold(
        body: Column(
          children: [
            UserInfo(user),
            Consumer(
              builder: (ctx, ref, _) {
                final filteredPosts = ref.watch(filteredPostsProvider);
                return PostList(filteredPosts);
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

### 2. 性能优化

dart

```dart
final userNameProvider = Provider<String>((ref) {
  return ref.watch(userProvider).select((user) => user.name);
});
```

------

## 六、最佳实践总结

|     场景     |            方案            |           代码示例           |
| :----------: | :------------------------: | :--------------------------: |
|   独立接口   |  FutureProvider + .family  | `userDetailProvider(userId)` |
|   组合状态   |       派生 Provider        |   `filteredPostsProvider`    |
|   临时状态   |    .autoDispose 修饰符     | `StateProvider.autoDispose`  |
| 复杂业务逻辑 | StateNotifier + Repository |        MVVM 架构实现         |
| 部分状态监听 |        select 方法         | `ref.watch(...).select(...)` |

------

**推荐工具链**：

- 使用 `flutter pub run build_runner watch` 持续生成代码
- 安装 Riverpod Snippets 插件加速开发

**官方文档**：Riverpod.dev