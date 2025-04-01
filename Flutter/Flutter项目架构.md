# 企业级 Flutter 项目架构方案

---

## 一、架构分层设计（Clean Architecture 演进版）

### 完整目录结构

```markdown

lib/
├── core/                       # 核心基础层
│   ├── di/                     # 依赖注入配置
│   ├── theme/                  # 主题管理
│   ├── utils/                  # 通用工具类
│   ├── routes/                 # 路由配置
│   └── network/                # 网络层核心配置
│       ├── dio_client.dart      # Dio单例配置
│       ├── interceptors/       # 拦截器集合
│       │   ├── logging_interceptor.dart
│       │   └── auth_interceptor.dart
│       ├── api_constants.dart   # API端点常量
│       └── exceptions/         # 网络异常处理
│           └── api_exception.dart
│
├── features/                   # 功能模块层
│   └── auth/                   # 认证模块
│       ├── domain/             # 领域层
│       │   ├── entities/       # 业务实体
│       │   ├── repositories/   # 抽象仓库接口
│       │   └── use_cases/      # 业务用例
│       │
│       ├── data/               # 数据层
│       │   ├── models/         # DTO模型
│       │   ├── datasources/    # 数据源
│       │   │   ├── auth_remote_datasource.dart
│       │   │   └── auth_local_datasource.dart  
│       │   └── repositories/   # 仓库实现
│       │
│       └── presentation/       # 表现层
│           ├── bloc/           # 状态管理
│           ├── widgets/        # 私有组件
│           └── screens/        # 页面组件
│
├── shared/                     # 共享资源层
│   ├── components/             # 全局通用组件
│   ├── services/               # 公共服务
│   └── localization/           # 国际化
│
└── main.dart                   # 应用入口
```



### 分层原则

1. **单向依赖**：表现层 → 领域层 → 数据层
2. **模块隔离**：每个 feature 模块可独立编译/测试
3. **抽象优先**：Repository 接口与实现分离

------

## 二、核心技术选型组合

### 1. 状态管理方案

#### 方案A：Riverpod（2.0+）

yaml

```yaml
dependencies:
  flutter_riverpod: ^2.3.6
  riverpod_annotation: ^2.3.6
dev_dependencies:
  riverpod_generator: ^2.3.6
```

**核心优势**：

- 完美解决 Provider 嵌套问题
- 支持自动回收资源和测试隔离
- 与代码生成深度集成

dart

```dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  Future<User> build() async => _loadInitialData();

  Future<void> login(String email, String password) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => _repo.login(email, password));
  }
}
```

#### 方案B：Bloc + Cubit

yaml

```yaml
dependencies:
  flutter_bloc: ^8.1.3
  bloc: ^8.1.2
  hydrated_bloc: ^9.1.2
```

**核心优势**：

- 明确的事件驱动架构
- 完善的状态生命周期管理
- 支持本地持久化（Hydrated Bloc）

dart

```dart
// Auth Cubit
class AuthCubit extends Cubit<AuthState> {
  final AuthRepository _repo;
  
  AuthCubit(this._repo) : super(AuthInitial());

  Future<void> login(String email, String password) async {
    emit(AuthLoading());
    try {
      final user = await _repo.login(email, password);
      emit(AuthAuthenticated(user));
    } catch (e) {
      emit(AuthError(e.toString()));
    }
  }
}

// 使用示例
BlocBuilder<AuthCubit, AuthState>(
  builder: (context, state) {
    return state.when(
      initial: () => Text('Ready'),
      loading: () => CircularProgressIndicator(),
      authenticated: (user) => ProfileView(user),
      error: (msg) => ErrorView(msg)
    );
  }
)
```

### 2. 路由管理

yaml

```yaml
dependencies:
  go_router: ^12.0.0
  auto_route: ^7.2.0
  auto_route_generator: ^7.2.0
```

dart

```dart
@AutoRouterConfig()
class AppRouter extends $AppRouter {
  @override
  List<AutoRoute> get routes => [
    AutoRoute(page: LoginRoute.page, path: '/login'),
    AutoRoute(page: HomeRoute.page, initial: true),
  ];
}

final goRouter = GoRouter(
  routes: $appRoutes,
  redirect: (context, state) {
    final isLoggedIn = authService.isAuthenticated;
    return (!isLoggedIn && state.location != '/login') ? '/login' : null;
  }
);
```

### 3. 依赖注入

yaml

```yaml
dependencies:
  get_it: ^7.6.4
  injectable: ^2.1.1
dev_dependencies:
  injectable_generator: ^2.1.1
```

dart

```dart
@singleton
class ApiService {
  final Dio dio;
  ApiService(this.dio);
}

@InjectableInit()
void configureDependencies() => getIt.init();
```

### 4. 网络层

yaml

```yaml
dependencies:
  dio: ^5.3.3
  retrofit: ^4.0.1
```

dart

```dart
dio.interceptors.addAll([
  LogInterceptor(requestBody: true),
  TokenInterceptor(),
  ErrorHandlerInterceptor()
]);
```

### 5. 本地存储

yaml

```yaml
dependencies:
  hive: ^2.2.3
  isar: ^3.1.3
```

dart

```dart
// Hive 简单存储
final settingsBox = await Hive.openBox('settings');
settingsBox.put('darkMode', true);

// Isar 高级查询
final users = await isar.users
  .filter().ageGreaterThan(18)
  .sortByName()
  .findAll();
```

------

## 三、企业级工具链

### 1. 代码规范检查

yaml

```yaml
dev_dependencies:
  flutter_lints: ^3.0.1
  custom_lint: ^0.6.0
```

### 2. 代码生成

yaml

```yaml
dev_dependencies:
  freezed: ^2.4.5
  json_serializable: ^6.7.1
  build_runner: ^2.4.6
```

### 3. 测试策略

|  测试类型  |       工具组合       |  覆盖率目标  |
| :--------: | :------------------: | :----------: |
|  单元测试  | bloc_test + mocktail |     ≥80%     |
| Widget测试 |    golden_toolkit    | 核心组件100% |
|  集成测试  |        patrol        | 关键路径覆盖 |

------

## 四、部署与监控

### 1. CI/CD 流程

```
开发提交 → SonarQube 扫描 → 单元测试 → 构建产物 → Firebase 分发 → 通知团队
```

### 2. 性能监控

| 监控维度 |         工具         |     关键指标      |
| :------: | :------------------: | :---------------: |
| 崩溃报告 | Firebase Crashlytics |   崩溃率 ≤ 0.1%   |
| 性能分析 |    Dart DevTools     |     FPS ≥ 58      |
| 网络质量 | Dio Logger + Sentry  | API成功率 ≥ 99.9% |

------

## 五、推荐架构组合

```
✅ 分层架构：Clean Architecture
✅ 状态管理：Riverpod 2.0 / Bloc + Cubit
✅ 路由导航：GoRouter + AutoRoute  
✅ 网络通信：Retrofit + Dio 拦截器链
✅ 本地存储：Isar（OLTP）/ Hive（KV存储）
✅ 依赖注入：GetIt + Injectable
✅ 代码规范：Lint + 自动化格式化
```

------

## 六、开发工作流

```
需求分析 → 创建 feature 模块
领域建模 → 定义实体/用例
代码生成 → 自动生成路由/模型
开发实现 → 遵循分层约束
质量验证 → 多维度测试
性能优化 → DevTools 分析
持续交付 → 自动化流水线
```

**多模块管理**：推荐使用 `melos` 管理跨模块依赖