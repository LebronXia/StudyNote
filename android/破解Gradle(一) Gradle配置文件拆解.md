## gradle是什么

- gradle是一个通用的构建工具，允许你构建任何软件，因为它很少假设你要构建什么或应该如何构建。最显著的限制是依赖关系管理目前只支持Maven和Ivy兼容的存储库和文件系统。
- grafle脚本使用了groovy或者Kotlin DSL ，gradle 使用 groovy 或者 kotlin 编写，不过目前还是 groovy 居多。
  那什么是 DSL 呢？DSL 也就是 Domain Specific Language 的简称，是为了解决某一类任务专门设计的计算机语言。
- gradle基于groovy编写，而groovy是基于jvm语言，所以本质上是面向对象的语言，面向对象语言的特点就是一切皆对象，所以，在 gradle 里，.gradle 脚本的本质就是类的定义，一些配置项的本质都是方法调用，参数是后面的 {} 闭包。

## 构建流程

![image-20211122165830528](../../../../Library/Application Support/typora-user-images/image-20211122165830528.png)

1. 将 .java 文件转换为 .class 文件，执行命令: javac xxx.java
2. Android 还会将 .class 文件转换为 .dex 文件: dx xxx.class
3. 打包成 apk: zip xxx.apk [需要打包的代码和资源]

## build配置文件

默认的创建Android工程都有什么：

![image-20211122153230639](../../../../Library/Application Support/typora-user-images/image-20211122153230639.png)

接下来大致介绍下每一个目或者文件的含义：

### settings.gradle

文件位于项目的根目录下，用于指定Gradle在构建应用时应将哪些模块包含在内，里面只包含以下内容：

```groovy
include ‘:app’
```

### rootproject/build.gradle

`build.gradle`负责整体项目的一些配置，对应的是`project`类。默认情况下，顶层构建文件使用 `buildscript` 代码块定义项目中所有模块共用的 Gradle 代码库和依赖项。

```groovy
//配置脚本的classpath
buildscript {
    ext.kotlin_version = "1.3.72"
  //配置仓库地址，后面的依赖都会在这里配置的地址查找
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
        google()
        jcenter()
    }
  //配置项目的依赖
    dependencies {
        classpath "com.android.tools.build:gradle:3.5.2"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

//配置项目及其子项目
allprojects {
    repositories {
        google()
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

> 有一点要提下，gradle默认是自顶向下执行，无论buildscript代码块在哪，它都会第一个执行。

需要配置项目全局属性的时候可以将额外的属性添加到顶层`build.gradle`文件内`ext`代码块中。

```groovy
buildscript {...}

allprojects {...}

ext {
    sdkVersion = 28
    supportLibVersion = "28.0.0"
    ...
}
```

### module/build.gradle

用于特定的子模块进行配置构建设置。可以通过配置这些构建设置提供自定义打包选项。

以app模块的build.gradle来看:

```groovy
// 引入 android gradle 插件
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}
apply plugin: 'MyPlugin'
apply plugin: HelloTransformPlugin

android {
    compileSdkVersion 30
    buildToolsVersion "30.0.2"

    defaultConfig {
        applicationId "com.example.gradlestudy"
        minSdkVersion 16
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    //指定java版本
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }

    //纬度
    flavorDimensions "tier"
    //不同的包
    productFLavors{
        free {
            dimension "tier"
            applicationId 'com.example.myapp.free'
        }

        paid {
            dimension "tier"
            applicationId 'com.example.myapp.paid'
        }
    }
}

// 项目需要的依赖
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.2.0'
    implementation 'androidx.appcompat:appcompat:1.1.0'
    implementation 'com.google.android.material:material:1.1.0'
    implementation 'androidx.constraintlayout:constraintlayout:1.1.3'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
}
```

**再说下`flavor`**:

在 android gradle plugin 3.x 之后，每个 flavor 必须对应一个 dimension，可以理解为 flavor 的分组，然后不同 dimension 里的 flavor 两两组合形成一个 variant。

![image-20211122171320719](../../../../Library/Application Support/typora-user-images/image-20211122171320719.png)

最后我们可以得出一个公式 【纬度1的产品风味数量】 * 【纬度2的产品风味数量】 * buildType数量 = 最终的APK变体数量。

### gradle/wrapper与gradlew.bat

gradlew.bat这个文件用来下载你特定版本的gradle然后执行，就不需要开发者在本地安装gradle了，客户以保证每个项目都使用自己期望的gradle版本。

gradlew 并没有直接启动 Gradle 而是启动 gradle-wrapper.jar，它会判断如果没有 Gradle 环境，从 gradle-wrapper.properties 中的 distributionUrl 中下载相应环境，并启动 Gradle。

`gradle/wrapper/gradle-wrapper.properties` 是一些gradlewrapper的配置，里面用的最多的是`distributionUrl`,是指可以执行gradle的下载地址和版本。

## 依赖管理

在gradle3.4里引入了新的依赖配置，可以到[Android 官方用户指南](https://developer.android.com/studio/build?hl=zh-cn#settings-file)看详细介绍如下：

![image-20211122162517419](../../../../Library/Application Support/typora-user-images/image-20211122162517419.png)

**举个栗子**： 假设 A 依赖 B，B 依赖 C。
 如果 B 对 C 使用 implementation 依赖，则 A 无法调用 C 的代码
 如果 B 对 C 使用 api 依赖，则 A 可以调用 C 的代码
 如果 B 对 C 使用 compileOnly 依赖，则 A 无法调用 C 的代码，且 C 的代码不会被打包到 APK 中
 如果 B 对 C 使用 runtimeOnly 依赖，则 A、B 无法调用 C 的代码，但 C 的代码会被打包到 APK 中

## task

看下面的代码，来对task有个简单的认识：

```groovy
task taskName{
  //执行在gradle生命周期的第二个阶段，即配置阶段执行
  println "This is myTask1"
}

//这里我们又定义了一个 taskName2，它依赖于 taskName
//也就是说，必须先执行完taskName才能去执行taskName2
task taskName2(depemdsOn: "taskName"){
  //加了 doFirst或doLast, 都执行在gradle生命周期的第三个阶段，即执行阶段执行
  doLast{
    println "android"
  }
}
```

执行的话，直接调`./gradlew taskName`。

对于 doFirst 与 doLast 这两个 Action，它们的作用分别如下所示：

- `doFirst`：表示 task 执行最开始的时候被调用的 Action。
- `doLast`：表示 task 将执行完的时候被调用的 Action。

需要注意的是，**doFirst 和 doLast 是可以被执行多次的**。

## gradle执行的生命周期

gradle构建分为三个阶段：
- **初始化阶段**
初始化阶段主要做的事情是决定有哪些项目需要被构建，然后为对应的项目创建 Project 对象
- **配置阶段**
  配置阶段主要做的事情是对上一步创建的项目进行配置，这时候会执行 build.gradle 脚本，并且会生成要执行的 task
- **执行阶段**
  执行阶段主要做的事情就是执行 task，进行主要的构建工作

Gradle提供了非常多的钩子供开发人员修改构建过程中的行为，为了方便说明，先看下面这张图。

![D27F48C8-4DB4-4907-A5D4-D71F8004A8E3](../../../../Library/Application Support/typora-user-images/D27F48C8-4DB4-4907-A5D4-D71F8004A8E3.png)

​								图片出自--[Gradle基础 - 构建生命周期和Hook技术](cnblogs.com/dasusu/p/8311324.html)

在阶段之间可以插入代码：

- 一二阶段之间:

  `settings.gradle` 的最后 

- 二三阶段之间:

  ```groovy
  gradle.beforeProject { project ->
    println 'apply plugin java for ' + project
    project.apply plugin: 'java'
  }
  //上面等价下面代码
  allprojects {
    beforeEvaluate { project ->
      println 'apply plugin java for ' + project
      project.apply plugin: 'java'
    }
  }
  ```

**参考**

[Gradle 爬坑指南 -- 导论](https://juejin.cn/post/6882178101191639053)

[百度技术-Gradle与Android构建入门](https://juejin.cn/post/6844904121217056782)

[【Android 修炼手册】Gradle 篇 -- Gradle 的基本使用](https://zhuanlan.zhihu.com/p/65249493)

[Gradle官方文档](https://docs.gradle.org/current/userguide/what_is_gradle.html)

[Android 官方用户指南](https://developer.android.com/studio/build?hl=zh-cn#settings-file)

