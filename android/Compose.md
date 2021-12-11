底部导航栏

BottomNavigation



[一个例子学会使用Jetpack Compose Modifier](https://blog.csdn.net/A_pyf/article/details/114002414)

[第二节-Arrangement](https://juejin.cn/post/6844904102577569806)



## 负面影响

- LaunchedEffect：在某个可组合项的作用域内运行挂起函数

- rememberCoroutineScope：获取组合感知作用域，以便在可组合项外启动协程

在非可组合项外启动协程，手动控制一个或多个协程的生命周期，可用`rememberCoroutineScope`

- rememberUpdatedState：在效应中引用某个值，该效应在值改变时不应重启

- DisposableEffect：需要清理的效应

  需要注册 `OnBackPressedCallback`才能监听在 `OnBackPressedDispatcher`上按下的返回按钮

使用 CompositionLocal 将数据的作用域限定在局部

- createRefs()  

引用是使用 `createRefs()` 或 `createRefFor()` 创建的，`ConstraintLayout` 中的每个可组合项都需要有与之关联的引用



1. [Bug小明 - Jetpack Compose 动画](https://juejin.cn/post/6971399722862903310)

2. [大佬们一起做的Compose UI 示例！！！强烈推荐](https://github.com/Gurupreet/ComposeCookBook)

3. [Compose图标库包](https://github.com/DevSrSouza/compose-icons)

4. [SmartToolFactory - 控件学习](https://github.com/SmartToolFactory/Jetpack-Compose-Tutorials)  (已经有提供的第三方控件)

5. [乐翁龙-Compose控件介绍](https://blog.csdn.net/u010976213/category_10622907.html)

6. ## **[Jetpack Compose 开发指南](https://gitee.com/Rickyal/compose-demo)**

7. [accompanist组件库](https://google.github.io/accompanist/)

8. [androiid官方 compose demo](https://github.com/android/compose-samples)

9. [compose 官方文档](https://developer.android.google.cn/reference/kotlin/androidx/compose/material/package-summary#BottomNavigation(androidx.compose.ui.Modifier,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.unit.Dp,kotlin.Function1))

10. [compose 官网](https://developer.android.google.cn/jetpack/compose/navigation)

buildAnnotatedAtring









  









