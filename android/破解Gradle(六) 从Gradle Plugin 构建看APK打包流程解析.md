目前，Android在进行构建APK，最常用到的就是Gradle打包。而要了解Android Apk打包的过程，就要深入了解Gradle Plugin的整个构建过程，在了解了之后，我们才能对Gradle Plugin开发游刃有余。

我们先了解一下APK内部的结构：

## 一、Android APK包结构

来看看一个正常的APK的结构。

我们可以打开build--outputs--apk--debug下的apk文件，就得到了下图，通常一个APK打包完之后，会有下面几个目录，用来存放不同的文件。

![image-20220116162815479](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220116162823.png)

**class.dex**

java代码通过javac转化成class文件，在通过dx文件转化成dex文件 。

**res**

保存了处理后的二进制资源文件。

**esoources.arsc**

保存了资源id名称以及资源对应的值/路径的映射。

**META-INF**

用来验证APK签名，其中三个重要的文件`MANIFEST.MT`，`CERT.SF`，`CERT.RSA`。

- **MANIFEST.MF**保存了所有文件对应的摘要
- **CERT.SF** 保存了MANIFEST.MF 中每条信息的摘要
- **CERT.RSA** 包含了对 CERT.SF 文件的签名以及签名用到的证书

**AndroidManifest.xml**

全局配置文件，这里是编译处理过的二进制的文件。

## 二、AppPlugin 构建流程

在分析前我们先编译下项目，直接左上方的编译按钮。可以看到会有以下输出：

```groovy
:android-gradle-plugin-source:preBuild UP-TO-DATE
:android-gradle-plugin-source:preDebugBuild
:android-gradle-plugin-source:compileDebugAidl
:android-gradle-plugin-source:compileDebugRenderscript
:android-gradle-plugin-source:checkDebugManifest
:android-gradle-plugin-source:generateDebugBuildConfig
:android-gradle-plugin-source:prepareLintJar UP-TO-DATE
:android-gradle-plugin-source:generateDebugResValues
:android-gradle-plugin-source:generateDebugResources
:android-gradle-plugin-source:mergeDebugResources
:android-gradle-plugin-source:createDebugCompatibleScreenManifests
:android-gradle-plugin-source:processDebugManifest
:android-gradle-plugin-source:splitsDiscoveryTaskDebug
:android-gradle-plugin-source:processDebugResources
:android-gradle-plugin-source:generateDebugSources
.......
```

从这里知道使用gradle构建，其实在使用一个个Task执行。在分析这些Task源码之前，我们跟下Gradle Plugin的构建过程。

为了能够查看Android Gradle Plugin的源码，我们需要在项目中添加android grade plugin依赖，如下所示：

```groovy
compileOnly 'com.android.tools.build:gradle:3.4.0'
```

好了可以开始看gradle源码了，但该从什么地方开始看起？

我们可以知道哦将一个module构建为一个Android项目，是在根目录下的build.gradle中配置了引入如下插件：

```groovy
apply plugin: 'com.android.application'
```

我们再上文中讲过自定义插件，要定义一个`xxx.properties`文件，里面声明插件的入口类。这里我们就知道了`android gradle plugin`的入口类，只要找到`com.android.application.properties`文件就可以了，然后可以看到内部标明了插件的实现类：

```groovy
implementation-class=com.android.build.gradle.internal.plugins.AppPlugin
```

这里定义了入口是`AppPlugin`，而AppPlugin是继承了`BasePlugin`:

```java
public class AppPlugin extends AbstractAppPlugin {
    .....
    //应用指定plugin
   @Override
    protected void pluginSpecificApply(@NonNull Project project) {
    }

    .....
      //获取一个扩展类，会提供一个相对应的android extension
  @Override
    @NonNull
    protected Class<? extends AppExtension> getExtensionClass() {
        return BaseAppModuleExtension.class;
    }

}
```

AppPlugin里面并没有做太多的操作，大部分工作还是在BasePlugin中做的。而主要的`apply`方法就是在`BasePlugin`中，在这个方法里面会进行一些准备工作，如下所示：

```java
public final void apply(@NonNull Project project) {
        CrashReporting.runAction(
                () -> {
                    basePluginApply(project);
                    pluginSpecificApply(project);
                });
    }
```

在apply方法里又调用了`basePluginApply`方法，如下所示：

```java
## BasePlugin.java
private void basePluginApply(@NonNull Project project) {
        // We run by default in headless mode, so the JVM doesn't steal focus.
        System.setProperty("java.awt.headless", "true");

        this.project = project;
        this.projectOptions = new ProjectOptions(project);
  			//检查gradle的版本号
        checkGradleVersion(project, getLogger(), projectOptions);
        DependencyResolutionChecks.registerDependencyCheck(project, projectOptions);

        project.getPluginManager().apply(AndroidBasePlugin.class);
				//路径检查
        checkPathForErrors();
  			//检查module名字是否合规
        checkModulesForErrors();

        PluginInitializer.initialize(project);
        ProfilerInitializer.init(project, projectOptions);
  			//记录方法耗时
        threadRecorder = ThreadRecorder.get();

        ......

        if (!projectOptions.get(BooleanOption.ENABLE_NEW_DSL_AND_API)) {
           // 配置工程
            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
                    project.getPath(),
                    null,
                    this::configureProject);
           // 配置 Extension
            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
                    project.getPath(),
                    null,
                    this::configureExtension);
           // 创建 Tasks
            threadRecorder.record(
                    ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
                    project.getPath(),
                    null,
                    this::createTasks);
        } else {
            ......
        }
    }
```

可以看到在`apply`方法里，`threadRecoirder.recode()`是记录最后一个参数的路径和执行的时间点，在前面做了一些插件检查操作以及对插件进行初始化与配置相关的信息，其实主要是做了下面这些事：

```java
// 配置项目，设置构建回调
this::configureProject
// 配置Extension
this::configureExtension
// 创建任务
this::createTasks
```

### 2.1 configurePoroject配置项目

这一阶段主要做了:

```java
## BasePlugin 
private void configureProject() {
        final Gradle gradle = project.getGradle();
        .....

          //AndroidSDK处理类
        sdkHandler = new SdkHandler(project, getLogger());
        if (!gradle.getStartParameter().isOffline()
                && projectOptions.get(BooleanOption.ENABLE_SDK_DOWNLOAD)) {
          //相关配置依赖的下载处理
            SdkLibData sdkLibData = SdkLibData.download(getDownloader(), getSettingsController());
            sdkHandler.setSdkLibData(sdkLibData);
        }

   		//创建AndroidBuilder
        AndroidBuilder androidBuilder =
                new AndroidBuilder(
                        project == project.getRootProject() ? project.getName() : project.getPath(),
                        creator,
                        new GradleProcessExecutor(project),
                        new GradleJavaProcessExecutor(project),
                        extraModelInfo.getSyncIssueHandler(),
                        extraModelInfo.getMessageReceiver(),
                        getLogger());
   			//创建 DataBindingBuilder 实例。
        dataBindingBuilder = new DataBindingBuilder();
        dataBindingBuilder.setPrintMachineReadableOutput(
                SyncOptions.getErrorFormatMode(projectOptions) == ErrorFormatMode.MACHINE_PARSABLE);

        if (projectOptions.hasRemovedOptions()) {
            androidBuilder
                    .getIssueReporter()
                    .reportWarning(Type.GENERIC, projectOptions.getRemovedOptionsErrorMessage());
        }

       ......

        // 强制使用不低于当前所支持的最小插件版本，否则会抛出异常。
        GradlePluginUtils.enforceMinimumVersionsOfPlugins(
                project, androidBuilder.getIssueReporter());

        // 应用 Java Plugin
        project.getPlugins().apply(JavaBasePlugin.class);

        DslScopeImpl dslScope =
                new DslScopeImpl(
                        extraModelInfo.getSyncIssueHandler(),
                        extraModelInfo.getDeprecationReporter(),
                        objectFactory);

        @Nullable
        FileCache buildCache = BuildCacheUtils.createBuildCacheIfEnabled(project, projectOptions);
   ......
     //给assemble任务添加甲描述
     project.getTasks()
                .getByName("assemble")
                .setDescription(
                        "Assembles all variants of all applications and secondary packages.");
  
     // project 执行完成结束后清除缓存操作
    gradle.addBuildListener(
                new BuildListener() {
                		.......
                    @Override
                    public void buildFinished(@NonNull BuildResult buildResult) {
                       
                        if (buildResult.getGradle().getParent() != null) {
                            return;
                        }
                        ModelBuilder.clearCaches();
                        sdkHandler.unload();
                        threadRecorder.record(
                                ExecutionType.BASE_PLUGIN_BUILD_FINISHED,
                                project.getPath(),
                                null,
                                () -> {
                                    WorkerActionServiceRegistry.INSTANCE
                                            .shutdownAllRegisteredServices(
                                                    ForkJoinPool.commonPool());
                                    Main.clearInternTables();
                                });
                      // 当任务执行完成时，清楚dex缓存
                        DeprecationReporterImpl.Companion.clean();
                    }
                });

    .....
 }
```

在这个方法里创建了`sdkHandle`在代码编译的时候会用到`android.jar`这个包。另外也创建了`AndroidBuilder`对象，这个对象是用来合并`manifest`和创建等作用。最后添加了`gradle`的生命周期监听，在`project`执行结束后清除了dex缓存。

### 2.2 configureExtension配置Extension

这个阶段就是配置extension的阶段，就是创建我们android块中的可配置对象。

1. 创建了`BuildType`、`ProductFlavor`、`SignIngConfig`三个类型的Container，接着就传入到了`createExtension`方法中。这里的`AppExtension`也就是`build.gradle`里的`android{}DSL闭包，来看下`createExtension`方法：

```java
## AbstractAppPlugin.java 
protected BaseExtension createExtension(
            @NonNull Project project,
            @NonNull ProjectOptions projectOptions,
            @NonNull GlobalScope globalScope,
            @NonNull SdkHandler sdkHandler,
            @NonNull NamedDomainObjectContainer<BuildType> buildTypeContainer,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavorContainer,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigContainer,
            @NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
            @NonNull SourceSetManager sourceSetManager,
            @NonNull ExtraModelInfo extraModelInfo) {
        return project.getExtensions()
                .create(
                        "android",
                        getExtensionClass(),
                        project,
                        projectOptions,
                        globalScope,
                        sdkHandler,
                        buildTypeContainer,
                        productFlavorContainer,
                        signingConfigContainer,
                        buildOutputs,
                        sourceSetManager,
                        extraModelInfo,
                        isBaseApplication);
    }
```

我们也可以看出android配置块是从Gradle构建中出来的。

2. 依次创建了一些管理类`variantFactory`、`taskManager`和`variantManager`。其中TaskManager是创建具体任务的管理类，`VariantFactory`是构建变体的工厂类，主要生成构建变体的对象。
3. 配置了`signingConfigContainer`、`buildTypeContainer`和`productFlavorContainer`的回调，每个配置的回调都会放入`variantManager`中进行管理。

```java
## BasePlugin.java
//将 whenObjectAdded callbacks 映射到 singingConfig 容器之中。     
signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

        buildTypeContainer.whenObjectAdded(
                buildType -> {
                    if (!this.getClass().isAssignableFrom(DynamicFeaturePlugin.class)) {
                        SigningConfig signingConfig =
                                signingConfigContainer.findByName(BuilderConstants.DEBUG);
                        buildType.init(signingConfig);
                    } else {
                        // initialize it without the signingConfig for dynamic-features.
                        buildType.init();
                    }
                    variantManager.addBuildType(buildType);
                });
				//将 whenObjectAdded callbacks 映射到 productFlavor 容器之中。
        productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);
```

4. 创建默认的debug签名，创建debug和release两个buildType。

```java
variantFactory.createDefaultComponents(
                buildTypeContainer, productFlavorContainer, signingConfigContainer);
```

真正的实现是在`ApplicationVariantFactory`中：

```java
 public void createDefaultComponents(
            @NonNull NamedDomainObjectContainer<BuildType> buildTypes,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavors,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigs) {
        // 必须首先创建签名配置，以便可以使用调试签名配置初始化构建类型“debug”。
        signingConfigs.create(DEBUG);
        buildTypes.create(DEBUG);
        buildTypes.create(RELEASE);
    }
```

### 2.3 createTasks创建 task

在`BasePlugin`的`createTask`方法里开始构建需要的Task，主要有两步，创建不依赖`flavor`的`task`和创建依赖配置项的配置的`task`。

```java
## BasePlugin.java
private void createTasks() {
  		//在evaluate之前创建Tasks
        threadRecorder.record(
                ExecutionType.TASK_MANAGER_CREATE_TASKS,
                project.getPath(),
                null,
                () -> taskManager.createTasksBeforeEvaluate());

  			//创建Android Tasks
        project.afterEvaluate(
                CrashReporting.afterEvaluate(
                        p -> {
                            sourceSetManager.runBuildableArtifactsActions();

                            threadRecorder.record(
                                    ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                    project.getPath(),
                                    null,
                                    this::createAndroidTasks);
                        }));
    }
```

1. **createTasksBeforeEvaluate()**

在这个方法里给容器注册了一系列的Task，包括uninstallAll，deviceCheck，connectedCheck，preBuild，extractProguardFiles，sourceSets，assembleAndroidTest，compileLint，lint，lintChecks，cleanBuildCacheresolveConfigAttr，consumeConfigAttr。

2. **createAndroidTasks()**

在这个方法主要是生成flavors相关数据，并根据flavor创建与之对应的Task实例并注册到Task当中。

```java
## BaePlugin.java
@VisibleForTesting
    final void createAndroidTasks() {
      ......
     // Project Path、CompileSdk、BuildToolsVersion、Splits、KotlinPluginVersion、FirebasePerformancePluginVersion 写入project配置
         ProcessProfileWriter.getProject(project.getPath())
                .setCompileSdk(extension.getCompileSdkVersion())
                .setBuildToolsVersion(extension.getBuildToolsRevision().toString())
                .setSplits(AnalyticsUtil.toProto(extension.getSplits()));

        String kotlinPluginVersion = getKotlinPluginVersion();
        if (kotlinPluginVersion != null) {
            ProcessProfileWriter.getProject(project.getPath())
                    .setKotlinPluginVersion(kotlinPluginVersion);
        }

        if (projectOptions.get(BooleanOption.INJECT_SDK_MAVEN_REPOS)) {
            sdkHandler.addLocalRepositories(project);
        }

      	// 创建应用的task
        List<VariantScope> variantScopes = variantManager.createAndroidTasks();
      
      ......
    }
```

我们主要来看下`variantManager`的`createAndroidTasks`方法：

```java
##   VariantManager.java
public List<VariantScope> createAndroidTasks() {
        variantFactory.validateModel(this);
        variantFactory.preVariantWork(project);

        if (variantScopes.isEmpty()) {
          //构建flavor变体
            populateVariantDataList();
        }

        taskManager.createTopLevelTestTasks(!productFlavors.isEmpty());

        for (final VariantScope variantScope : variantScopes) {
          //为变体数据创建相对应的task
            createTasksForVariantData(variantScope);
        }

        taskManager.createSourceSetArtifactReportTask(globalScope);

        taskManager.createReportTasks(variantScopes);

        return variantScopes;
    }
```

首先会判断`variantScopes`为空，就会调用`populateVariantDataList`方法，在这个方法里会根据flavor和dimension创建对应的组合，存放在`flavorComboList，`最后会调用`createVariantDataForProductFlavors`方法。

```java
## VariantManager.java
public void populateVariantDataList() {
  //
  List<String> flavorDimensionList = extension.getFlavorDimensionList();
  ......
    //迭代获取productFlavor数组
    Iterable<CoreProductFlavor> flavorDsl =
                    Iterables.transform(
                            productFlavors.values(),
                            ProductFlavorData::getProductFlavor);
  
  // 创建 flavor 和 dimension 的组合
    List<ProductFlavorCombo<CoreProductFlavor>> flavorComboList =
                    ProductFlavorCombo.createCombinations(
                            flavorDimensionList,
                            flavorDsl);

  //为每个组合创建VariantData
            for (ProductFlavorCombo<CoreProductFlavor>  flavorCombo : flavorComboList) {
                //noinspection unchecked
                createVariantDataForProductFlavors(
                        (List<ProductFlavor>) (List) flavorCombo.getFlavorList());
            }
  ......
}
```

再来看下`createVariantDataForProductFlavors`，最终调用的方法是`createVariantDataForProductFlavorsAndVariantType`，在这个方法里将创建得到的`VariantData`,并且加入到了variantScopes集合中，这里我们就将所有的构建变体集合到了 variantScopes 中。

```java
## VariantManager.java
private void createVariantDataForProductFlavorsAndVariantType(
            @NonNull List<ProductFlavor> productFlavorList, @NonNull VariantType variantType) {
  ......
  BaseVariantData variantData =
                        createVariantDataForVariantType(
                                buildTypeData.getBuildType(), productFlavorList, variantType);
                addVariant(variantData);
  ......
}

public void addVariant(BaseVariantData variantData) {
        variantScopes.add(variantData.getScope());
    }
```

接着再来看下`createTasksForVariantData`方法，创建完 variant 数据，就要给每个 variantData 创建对应的 task。

```java
## VariantManager
public void createTasksForVariantData(final VariantScope variantScope) {
        final BaseVariantData variantData = variantScope.getVariantData();
        final VariantType variantType = variantData.getType();
        final GradleVariantConfiguration variantConfig = variantScope.getVariantConfiguration();

  		  //创建 assembleXXXTask
        taskManager.createAssembleTask(variantData);
        if (variantType.isBaseModule()) {
          //如果 variantType 是 base moudle，则会创建相应的 bundle Task
            taskManager.createBundleTask(variantData);
        }
   if (variantType.isTestComponent()) {
     		......
   } else {
     // 根据 variantData 创建一系列任务
     taskManager.createTasksForVariantScope(variantScope);
   }
}
  
```

首先会创建`Assemble Task`，来看下`createAssembleTask`方法:

```java
## TaskManager  
public void createAssembleTask(@NonNull final BaseVariantData variantData) {
        final VariantScope scope = variantData.getScope();
  			//最终会在TaskContainer中进行注册
        taskFactory.register(
          			//得到assemble xxx
                getAssembleTaskName(scope, "assemble"),
                null /*preConfigAction*/,
                task -> {
                    task.setDescription(
                            "Assembles main output for variant "
                                    + scope.getVariantConfiguration().getFullName());
                },
                taskProvider -> scope.getTaskContainer().setAssembleTask(taskProvider));
    }
```

然后回到`createTasksForVariantData`方法里，再来看下`createTasksForVariantScope`方法，它是个抽象方法，最终会执行到`ApplicationTaskManager` 的 `createTasksForVariantScope` 方法。

```java
@Override
public void createTasksForVariantScope(@NonNull final VariantScope variantScope) {
        createAnchorTasks(variantScope);
        createCheckManifestTask(variantScope);   //检测 manifest
        handleMicroApp(variantScope);
        createDependencyStreams(variantScope);
        createApplicationIdWriterTask(variantScope);    // application id 
        taskFactory.register(new MainApkListPersistence.CreationAction(variantScope));
        createBuildArtifactReportTask(variantScope);
        createMergeApkManifestsTask(variantScope);   // 合并 manifest
        createGenerateResValuesTask(variantScope);
        createRenderscriptTask(variantScope);
        createMergeResourcesTask(variantScope, true, ImmutableSet.of());   // 合并资源文件
        createShaderTask(variantScope);
        createMergeAssetsTask(variantScope);
        createBuildConfigTask(variantScope);
        createApkProcessResTask(variantScope);    // 处理资源
        createProcessJavaResTask(variantScope);
        createAidlTask(variantScope);      // 处理 aidl
        createMergeJniLibFoldersTasks(variantScope);    // 合并 jni
        createDataBindingTasksIfNecessary(variantScope, MergeType.MERGE);    // 处理 databinding
        TaskProvider<? extends JavaCompile> javacTask = createJavacTask(variantScope);
        addJavacClassesStream(variantScope);
        setJavaCompilerTask(javacTask, variantScope);
        createPostCompilationTasks(variantScope);   //处理 Android Transform
        createValidateSigningTask(variantScope);
        taskFactory.register(new SigningConfigWriterTask.CreationAction(variantScope));
        createPackagingTask(variantScope, null /* buildInfoGeneratorTask */);   // 打包 apk
        createConnectedTestForVariant(variantScope);
  .......
}
```

可以看到在这个方法里创建了适用于应用构建的一些列Tasks，到这里`gradle Plugin`的构建过程也算跟完了。

下面我们通过`assembleDebug `的打包流程来分析一下这些 `Tasks`。

## 三、App打包流程

### 3.1 打包流程浅析

开始前我们先回顾下apk的打包流程，可以先看下[Android](https://developer.android.com/studio/build)官网给出的图：

![image-20220118001131512](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220118001133.png)

官网上给出的图比较简略，只能看出个大概逻辑，我们再来看下更详细的打包过程的细节。

![image-20220118102641831](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220118102644.png)

可以概括为以下七个步骤：

1. 通过`aapt`打包res资源文件，生成`R.java`、`resources.arsc`和`res`文件
2. 处理`.aidl`文件，生成对应的`Java`接口文件
3. 通过`Java Compier`编译`R.java`、Java接口文件、java源文件，生成.class文件
4. 通过`dex`命令，将`.class`文件和第三方库中的`.class`文件处理生成`classes.dex`
5. 通过`apkbuilder`工具，将`aapt`(在android gradle plugin 3.0.0之后使用AAPT2替代了AAPT)生成的`resources.arsc`和`res`文件、`assets`文件和`classes.dex`一起打包生成apk
6. 通过`jarsigner签名工具，对上面的apk进行debug和release签名
7. 通过`zipalign`工具，将签名后的`apk`进行对齐处理

到了这里，我们对打包流程有了大致的了解，那么以Task纬度来看apk打包又会是什么流程呢？

### 3.2 Task纬度分析打包流程

我们可以通过下面的命令来获取打包一个Debug Apk包，看下打包一次都涉及到了哪些Task。

```java
./gradlew assembleDebug --console=plain
```

可以看到以下的输出结果：

```java
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:compileDebugAidl NO-SOURCE
> Task :app:checkDebugManifest UP-TO-DATE
> Task :app:generateDebugBuildConfig UP-TO-DATE
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:mainApkListPersistenceDebug UP-TO-DATE
> Task :app:generateDebugResValues UP-TO-DATE
> Task :app:generateDebugResources UP-TO-DATE
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
> Task :app:processDebugManifest UP-TO-DATE
> Task :app:mergeDebugResources UP-TO-DATE
> Task :app:processDebugResources UP-TO-DATE
> Task :app:compileDebugKotlin UP-TO-DATE
> Task :app:prepareLintJar UP-TO-DATE
> Task :app:generateDebugSources UP-TO-DATE
> Task :app:javaPreCompileDebug UP-TO-DATE
> Task :app:compileDebugJavaWithJavac UP-TO-DATE
> Task :app:compileDebugSources UP-TO-DATE
> Task :app:mergeDebugShaders UP-TO-DATE
> Task :app:compileDebugShaders UP-TO-DATE
> Task :app:generateDebugAssets UP-TO-DATE
> Task :app:mergeDebugAssets UP-TO-DATE
> Task :app:validateSigningDebug UP-TO-DATE
> Task :app:signingConfigWriterDebug UP-TO-DATE
> Task :app:checkDebugDuplicateClasses UP-TO-DATE
> Task :app:transformClassesWithDexBuilderForDebug UP-TO-DATE
> Task :app:transformDexArchiveWithExternalLibsDexMergerForDebug UP-TO-DATE
> Task :app:transformDexArchiveWithDexMergerForDebug UP-TO-DATE
> Task :app:mergeDebugJniLibFolders UP-TO-DATE
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:transformResourcesWithMergeJavaResForDebug UP-TO-DATE
> Task :app:transformNativeLibsWithMergeJniLibsForDebug UP-TO-DATE
> Task :app:packageDebug UP-TO-DATE
> Task :app:assembleDebug UP-TO-DATE
```

从之前的App Pugin构建流程分析我们已经知道了，task的实现基本可以在`TaskManager`中找到，而要创建出task的方法主要有两个，分别是`Taskmanager.createTasksBeforeEvaluate`和`ApplicationTaskManager.createTasksForVariantScope()`。

下面我们再来看`看assemebleDebug`打包流程中所需的各个Task所对应的实现类和含义，下面就列出了各个task的作用以及实现类：

| Task                                                 | 对应实现类                          | 作用                                                         |
| ---------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| preBuild                                             |                                     | 空 task，只做锚点使用                                        |
| preDebugBuild                                        |                                     | 空 task，只做锚点使用，与 preBuild 区别是这个 task 是 variant 的锚点 |
| compileDebugAidl                                     | AidlCompile                         | 处理 aidl                                                    |
| compileDebugRenderscript                             | RenderscriptCompile                 | 处理 renderscript                                            |
| checkDebugManifest                                   | CheckManifest                       | 检测 manifest 是否存在                                       |
| generateDebugBuildConfig                             | GenerateBuildConfig                 | 生成 BuildConfig.java                                        |
| prepareLintJar                                       | PrepareLintJar                      | 拷贝 lint jar 包到指定位置                                   |
| generateDebugResValues                               | GenerateResValues                   | 生成 resvalues，generated.xml                                |
| generateDebugResources                               |                                     | 空 task，锚点                                                |
| mergeDebugResources                                  | MergeResources                      | 合并资源文件                                                 |
| createDebugCompatibleScreenManifests                 | CompatibleScreensManifest           | manifest 文件中生成 compatible-screens，指定屏幕适配         |
| processDebugManifest                                 | ProcessApplicationManifest          | 合并 manifest 文件                                           |
| splitsDiscoveryTaskDebug                             | SplitsDiscovery                     | 生成 split-list.json，用于 apk 分包                          |
| processDebugResources                                | LinkApplicationAndroidResourcesTask | aapt 打包资源                                                |
| generateDebugSources                                 |                                     | 空 task，锚点                                                |
| javaPreCompileDebug                                  | JavaPreCompileTask                  | 生成 annotationProcessors.json 文件                          |
| compileDebugJavaWithJavac                            | AndroidJavaCompile                  | 编译 java 文件                                               |
| compileDebugNdk                                      | NdkCompile                          | 编译 ndk                                                     |
| compileDebugSources                                  |                                     | 空 task，锚点使用                                            |
| mergeDebugShaders                                    | MergeSourceSetFolders               | 合并 shader 文件                                             |
| compileDebugShaders                                  | ShaderCompile                       | 编译 shaders                                                 |
| generateDebugAssets                                  |                                     | 空 task，锚点                                                |
| mergeDebugAssets                                     | MergeSourceSetFolders               | 合并 assets 文件                                             |
| transformClassesWithDexBuilderForDebug               | DexArchiveBuilderTransform          | class 打包 dex                                               |
| transformDexArchiveWithExternalLibsDexMergerForDebug | ExternalLibsMergerTransform         | 打包三方库的 dex，在 dex 增量的时候就不需要再 merge 了，节省时间 |
| transformDexArchiveWithDexMergerForDebug             | DexMergerTransform                  | 打包最终的 dex                                               |
| mergeDebugJniLibFolders                              | MergeSouceSetFolders                | 合并 jni lib 文件                                            |
| transformNativeLibsWithMergeJniLibsForDebug          | MergeJavaResourcesTransform         | 合并 jnilibs                                                 |
| transformNativeLibsWithStripDebugSymbolForDebug      | StripDebugSymbolTransform           | 去掉 native lib 里的 debug 符号                              |
| processDebugJavaRes                                  | ProcessJavaResConfigAction          | 处理 java res                                                |
| transformResourcesWithMergeJavaResForDebug           | MergeJavaResourcesTransform         | 合并 java res                                                |
| validateSigningDebug                                 | ValidateSigningTask                 | 验证签名                                                     |
| packageDebug                                         | PackageApplication                  | 打包 apk                                                     |
| assembleDebug                                        |                                     | 空 task，锚点                                                |

在`gradle Plugin`中的Task任务会有三种，一种是非增量task，一种是增量task，一种是transform task。

1. **非增量task**

   看 @TaskAction 注解的方法

2. **增量task**

   首先看`isIncremental`方法是否支持增量，然后再看`doFullTaskAction`方法。如果是支持增量，还要看`doIncrementalTaskAction`方法（增量的执行操作方法）

3. **transform task**

   直接看`transform`方法的实现

接下来我们会对其中几个重要的Task的实现进行详细分析。

> **小知识点：**当我们很难找到task所对应的实现类的时候，可以在build.gradle中加入，就可以在命令窗口中输出来了。
>
> ```java
> gradle.taskGraph.whenReady {
>     it.allTasks.each{ task ->
>         println "${task.name} : ${task.class.name - "_Decorated"}"
>     }
> }
> ```
>
> 

### 3.3 重点Task实现分析

#### 3.3.1 compileDebugAidl(编译.aidl文件)

`compileDebugAidl`的实现类是`AidlCompile，将.aidl文件通过aidl工具转换成编译器能够处理的Java接口文件。

> 调用链路：AidlCompile.doFullTaskAction -> workers.submit(AidlCompileRunnable.class, new AidlCompileParams(dir, processor)) -> DirectoryWalker.walk() ->  action.call() -> AidlProcessor.call() 

由于是增量Task，直接来看下`doFullTaskAction`方法:

```java
@TaskAction
    public void doFullTaskAction() throws IOException {
      ......
        AidlProcessor processor =
                    new AidlProcessor(
                            aidl,
                            target.getPath(IAndroidTarget.ANDROID_AIDL),
                            fullImportList,
                            sourceOutputDir,
                            packagedDir,
                            packageWhitelist,
                            new DepFileProcessor(),
                            processExecutor,
                            new LoggedProcessOutputHandler(new LoggerWrapper(getLogger())));

      		//会生成一个processor对象，并交由AidlCompileRunnable
      		//在这个线程里异步执行
            for (File dir : sourceFolders) {
                workers.submit(AidlCompileRunnable.class, new AidlCompileParams(dir, processor));
            }
            workers.close();
    }
```

#### 3.3.2 generateDebugBuildConfig（生成BuildConfig文件）

在AppplicationTaskManager中createBuildConfigTask 方法被调用会生成一个BuildConfig文件。再来看下`createBuildCoonfigTask`方法，会再里面创建出`GenerateBuildConfig`对象，主要看下`TaskAction`标记的代码：

```java
## GenerateBuildConfig.java
    @TaskAction
    void generate() throws IOException {
        
        File destinationDir = getSourceOutputDir();
        FileUtils.cleanOutputDir(destinationDir);

        BuildConfigGenerator generator = new BuildConfigGenerator(
                getSourceOutputDir(),
                getBuildConfigPackageName());

        // 添加默认的属性，包括 DEBUG，APPLICATION_ID，FLAVOR，VERSION_CODE，VERSION_NAME
        generator
                .addField(
                        "boolean",
                        "DEBUG",
                        isDebuggable() ? "Boolean.parseBoolean(\"true\")" : "false")
                .addField("String", "APPLICATION_ID", '"' + appPackageName.get() + '"')
                .addField("String", "BUILD_TYPE", '"' + getBuildTypeName() + '"')
                .addField("String", "FLAVOR", '"' + getFlavorName() + '"')
                .addField("int", "VERSION_CODE", Integer.toString(getVersionCode()))
                .addField(
                        "String", "VERSION_NAME", '"' + Strings.nullToEmpty(getVersionName()) + '"')
                .addItems(getItems());   //添加自定义的属性

        List<String> flavors = getFlavorNamesWithDimensionNames();
        int count = flavors.size();
        if (count > 1) {
            for (int i = 0; i < count; i += 2) {
                generator.addField(
                        "String", "FLAVOR_" + flavors.get(i + 1), '"' + flavors.get(i) + '"');
            }
        }
			//内部调用javaWriter生成文件
        generator.generate();
    }
```

首先创建了`BuildConfigGenerator`，又添加了DEBUG，APPLICATION_ID，FLAVOR，VERSION_CODE，VERSION_NAME，以及添加了自定义属性，最后调用`JavaWriter`生成`BuildConfig.java`文件。

#### 3.3.3 mergeDebugResources（合并资源文件）

`mergeDebugResources`对应的实现类是`MergeResources`。

将res目录下的资源整合在一起，通过aapt2(android资源打包工具)进行编译打包。aapt2将资源编译的任务分成编译和连接，会生成flat文件，将资源文件编译成二进制的文件。

> 调用链路：MergeResources.doFullTaskAction() -> merger.mergeData() -> DataMerger.mergeData() -> MergedResourceWriter.end() -> ResourceCompiler.submitCompile -> AaptV2CommandBuilder.makeCompileCommand()

回到源码可以看到MergeResources是支持增量编译的，所以我们来看下它的`doFullTaskAction()`方法：

1. 获取到所有的`resourceSet`,而它包括了项目中所有的res资源，并遍历放入了`ResourceMerger`集合中

	```java
   List<ResourceSet> resourceSets = getConfiguredResourceSets(preprocessor);
   ......
    for (ResourceSet resourceSet : resourceSets) {
                resourceSet.loadFromFiles(getILogger());
                merger.addDataSet(resourceSet);
            }
	```

2. 调用`ResourceMerger`整合资源

   ```java
   merger.mergeData(writer, false /*doCleanUp*/);
	```

3. 最终调用`DataMerger`的`mergeData`方法，在里面调用了`MergedResourceWriter`的start()，addItem()，end()方法，主要来看下addItem方法，分别添加到`ValuesResMap`和`CompileResourceRequests`容器当中：

	```java
    @Override
       public void addItem(@NonNull final ResourceMergerItem item) throws ConsumerException {
       		......
         if (type == ResourceFile.FileType.XML_VALUES) {
               //xml文件添加
               mValuesResMap.put(item.getQualifiers(), item);
           } else {
           ......
             //收集新的处理请求
               mCompileResourceRequests.add(
                           new CompileResourceRequest(file, getRootFolder(), folderName));
           }
       }
   ```
   
4. 最后在来看下`MergedResourceWriter`的end方法，看它的父类实现，会调用到`postWriteAction`方法，遍历资源文件，创建request对象，最后添加进`ResourceCompiler`中。

   ```java
   ##  MergeResourceWriter
   @Override
       protected void postWriteAction() throws ConsumerException {
       	......
       	 // now write the values files.
           for (String key : mValuesResMap.keySet()) {
           	......
           	CompileResourceRequest request =
                               new CompileResourceRequest(
                                       outFile,
                                       getRootFolder(),
                                       folderName,
                                       pseudoLocalesEnabled,
                                       crunchPng,
                                       blame != null ? blame : ImmutableMap.of());
            ......
            mResourceCompiler.submitCompile(request);
           }
       }
   ```

   还是回到`MergedResourceWriter`的`end`方法，又看到了这个方法，资源最后会在end方法里被处理。

   ```java
   mResourceCompiler.submitCompile(
                               new CompileResourceRequest(
                                       fileToCompile,
                                       request.getOutputDirectory(),
                                       request.getInputDirectoryName(),
                                       pseudoLocalesEnabled,
                                       crunchPng,
                                       ImmutableMap.of(),
                                       request.getInputFile()));
   ```

   最终调用`AaptV2CommandBuilder.makeCompileCommand() `方法生成aapt2命令去处理资源，处理完以后生成 xxx.xml.flat 格式。

   > 在获取 resourceSets 的时候，使用的是修改后的文件，图片转webp格式的插件也可以放在这个task前面。

#### 3.3.4 processDebugResources（打包资源文件）

`processDebugResources` 对应的实现类是`LinkApplicationAndroidResourcesTask`。

> 调用链路：LinkApplicationAndroidResourcesTask.doFullTaskAction() -> AaptSplitInvoker.run() -> invokeAaptForSplit() -> AndroidBuilder.processResources() ->aapt.link()

最终会调用`AaptV2CommandBuilder.makeLinkCommand()`方法打包资源并生成R.java文件和.ap_资源。

#### 3.3.5 processDebugManifest（合并 manifest 文件）

`processDebugManifest`对应的实现类是`ProcessApplicationManifest`

最终合并AndroidManifest.xml文件。

#### 3.3.6 transformClassesWithDexBuilderForDebug（class打包dex）

`transformClassesWithDexBuilderForDebug`的实现类是`TransformTask`，而真正实现将.class文件转成.dex文件的是`DexArchiveBuilderTransform`。

> 调用链路：DexArchiveBuilderTransform.transform -> convertToDexArchive  -> launchProcessing -> dexArchiveBuilder.convert -> DxDexArchiveBuilder.dex -> CfTranslator.translate

来看下它的`transform`方法：

```java
public void transform(@NonNull TransformInvocation transformInvocation)
            throws TransformException, IOException, InterruptedException {
            ......
              
                for (DirectoryInput dirInput : input.getDirectoryInputs()) {
                    logger.verbose("Dir input %s", dirInput.getFile().toString());
                  //自己的会调用convertToDexArchive
                    convertToDexArchive(
                            transformInvocation.getContext(),
                            dirInput,
                            outputProvider,
                            isIncremental,
                            bootclasspathServiceKey,
                            classpathServiceKey,
                            additionalPaths);
                }
						
                for (JarInput jarInput : input.getJarInputs()) {
                    logger.verbose("Jar input %s", jarInput.getFile().toString());
                   ......

                  //第三方的会调用processJarInput
                    List<File> dexArchives =
                            processJarInput(
                                    transformInvocation.getContext(),
                                    isIncremental,
                                    jarInput,
                                    outputProvider,
                                    bootclasspathServiceKey,
                                    classpathServiceKey,
                                    additionalPaths,
                                    cacheInfo);
                  ......
            }
```

`processJarInput`方法内部最终也还是会调用`convertToDexArchive`方法，主要是会对dir以及jar的后续处理。在里面有会调用`launchProcessing`方法。

```java
private static void launchProcessing(
            @NonNull DexConversionParameters dexConversionParameters,
            @NonNull OutputStream outStream,
            @NonNull OutputStream errStream,
            @NonNull MessageReceiver receiver)
            throws IOException, URISyntaxException {
  ......
    //是个抽象类，最后会根据d8工具或是dx工具来处理，来完成从.class代码转成.dex代码
    dexArchiveBuilder.convert(
                    entries,
                    Paths.get(new URI(dexConversionParameters.output)),
                    dexConversionParameters.isDirectoryBased());
  ......
}
```

`dexArchiveBuilder.convert`有两个实现子类，`D8DexArchiveBuilder` 和 `DxDexArchiveBuilder`，分别是调用 d8 和 dx 去打 dex。

最后看下该图就是上面所最终完成的效果：

![image-20220119003112940](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220119003115.png)



## 小结

到这里也就结束了，我们不妨再回忆下`gradle plugin`构建中发生了什么事情，以及apk打包过程中会涉及到哪些task任务，如果都明白了，也算看懂了。

## 参考

[【Android 修炼手册】常用技术篇 -- 聊聊 Android 的打包](https://juejin.cn/post/6844903910734299149)

[【Android 修炼手册】Gradle 篇 -- Android Gradle Plugin 主要流程分析](https://juejin.cn/post/6844903844749508622#heading-7)

[补齐Android技能树——从AGP构建过程到APK打包过程](https://juejin.cn/post/6963527524609425415)

[【灵魂七问】深度探索 Gradle 自动化构建技术（五、Gradle 插件架构实现原理剖析 — 下）](https://juejin.cn/post/6844904142717075469#heading-41)

[Android Gradle Plugin 源码解析（上）](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492752&idx=1&sn=1d1ad65c63667d96b72452a492cbde58&chksm=8eec86efb99b0ff9378916d061f70d56bc1c27c795ef5c36cd5692886c3cbb1109636951a5c3&scene=38#wechat_redirect)



