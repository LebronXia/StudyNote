如果你想在编译期间搞事情，如常用的有无痕埋点，方法耗时统计和组件通信中自动注入等等，就要来学习字节码插桩的技术。而所谓字节码插桩技术其实就是修改已经编译好的class文件，在里面添加自己的字节码，然后打出的包就是修改后的class文件。在动手开发之前，还需要了解如何`自定义gradle插件`，以及如何`自定义Transform`，下面我们来看看具体做法。

## 一、 Gradle插件

在[Gradle官方文档](https://docs.gradle.org/current/userguide/custom_plugins.html)里目前定义插件的方式有 三种：

1. `脚本插件`：直接在构建脚本中直接写插件的代码，编译器会自动将插件编译并添加到构建脚本的classpath中。
2. `buildSrc project`：执行Gradle时 会把根目录下的buildSrc目录作为插件源码目录进行编译，编译后会把结果加入到构建脚本的classpath中，对于整个项目是可用的。
3. `Standalone project`：可以在独立项目中开发插件，然后将项目达成jar包，发布到本地或者mave服务器上。

**实例代码可以参考 [GradleTestDemo](https://github.com/LebronXia/GradleTestDemo)**

### 1.1 直接在build.gradle文件中实现

```groovy
//应用插件
apply plugin: CustomPluginA

//自定义插件示例
class CustomPluginA implements Plugin<Project> {

    @Override
    void apply(Project target) {
        println 'Hello gradle!'
    }
}
```

这种方式在构建脚本之外是不可以见的，所以只有在定义该插件的gradle脚本里才可以引用改插件。

### 1.2 在默认目录buildSrc中实现

buildSrc目录是gradle默认的目录之一，该目录会在构建时自动的进行编译打包，所以在这里面不需要任何额外的配置，就可以直接被其他模块中的gradle脚本所引用。

 1. 创建的目录结构

![image-20220226155104045](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220226155117.png)
2. 将项目中的build.gradle中的所有配置去掉哦，并配置groovy、resoources为源码目录以及相关依赖：
 ```groovy
 buildscript {
    ext {
        kotlin_version = '1.5.31'
        apg_Version = '3.4.0'
        booster_version = '4.0.0'
    }
    repositories {
        mavenCentral()
        google()
        jcenter()
    }
   
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath "com.android.tools.build:gradle:$apg_Version"
    }
}

apply plugin: 'maven'
apply plugin: 'maven-publish'
apply plugin: 'java'
apply plugin: 'groovy'
apply plugin: 'kotlin'
apply plugin: 'kotlin-kapt'
apply plugin: 'kotlin-android-extensions'

repositories {
    mavenCentral()
    google()
    jcenter()
}

dependencies {
    implementation gradleApi()
    implementation localGroovy()
  	//操作的工具类
    implementation "commons-io:commons-io:2.6"

    // Android DSL  Android编译的大部分gradle源码
    implementation 'com.android.tools.build:gradle:3.4.0'
   
    //ASM
    implementation 'org.ow2.asm:asm:7.1'
    implementation 'org.ow2.asm:asm-util:7.1'
    implementation 'org.ow2.asm:asm-commons:7.1'
}
 ```

gradle插件是可以使用java，groovy，kotlin编写 ，所以你可以根据自己的需要引入相关的依赖。

3. 在main目录下新建resources/MATE-INF/gradle-plugins目录：

   在里面新建“HelloPlugin.properties”文件，其中“HelloPlugin”是可以随意定义的名称，而这个名称也就是你的插件名称。然后在引用该插件时你可以通过apply plugin: 'HelloPlugin'的方式来引用。

   `HelloPlugin.properties`文件中的内容就是：

   ```groovy
   implementation-class=transform.hello.HelloTransformPlugin
   ```

   另外定义在`buildSrc`下面的插件也可以直接用`apply plugin: HelloPlugin`来引入，而HelloPlugin就是你定义的plugin的类名了。

   > 注意：格式一定要写成resources/MATE-INF/gradle-plugins这样的三级目录，有些同学操作的时候碰到自定义的plugin找不到就是因为在直接复制目录地址的时候，由于编译器缩写的关系，目录地址变成了resources/MATE-INF.gradle-plugins。

4. 自定义gradle插件：

   在`transform`目录下创建`HelloTransformPlugin`类，并实现Plugin接口。

   ```groovy
   class HelloTransformPlugin implements Plugin<Project> {
   
       @Override
       void apply(Project project) {
           println "Hello TransformPlugin"
         
          //将Extension注册给Plugin
           def extension = project.extensions.create("custom", CustomExtension)
         
           //注册方式1  AppExtension就是application plugin
           AppExtension appExtension = project.extensions.getByType(AppExtension)
           appExtension.registerTransform(new HelloTransform())
           //注册之后会在TransformManager#addTransform中生成一个task.
   
           //注册方式2
           //project.android.registerTransform(new HelloTransform())
       }
   }
   ```

   你会看到这里多了个`Transform类`也是接下来我们要说的。另外其中的 CustomExtension 是自定义属性类，可以通过主项目的 build.gradle 文件传值，这样就可以在脚本中去扩展属性：

   ```groovy
   class CustomExtension {
       String extensionArgs = ""
   }
   ```

   然后在主项目的`build.gradle`命名要与注册时保持一致：

   ```groovy
   custom{
       extensionArgs = "我是参数"
   }
   ```

   在 `project.extensions.create` 方法的内部其实质是 通过 `project.extensions.create() `方法来获取在 `custom` 闭包中定义的内容并通过反射将闭包的内容转换成一个 `CustomExtension` 对象。

### 1.3 在独立项目开发中实现

   这种方式基本跟第二种相似，不过要引入这个插件的话要先把它发布到本地或者mave服务器上。

1. 修改 build.gradle 的内容，增加上传到本地的代码，可以如下这样修改：

   ```groovy
   apply plugin: 'groovy'
   apply plugin: 'java'
   apply plugin: 'maven'
   
   repositories {
       jcenter()
   }
   
   uploadArchives {
       repositories.mavenDeployer {
    				//指定maven的仓库url，IP+端口+目录
   //        repository(url: "http://localhost:8081/nexus/content/repositories/releases/") {
   //            //填写你的Nexus的账号密码
   //            authentication(userName: "admin", password: "123456")
   //        }
         
           // 配置本地仓库路径，这里是项目的根目录下的maven目录中
           repository(url: uri('../repo'))
           // 唯一标识 一般为模块包名 也可其他
           pom.groupId = "com.xiam.plugin"
           // 项目名称（一般为模块名称 也可其他
           pom.artifactId = "startplugin"
           // 发布的版本号
           pom.version = "1.0.0"
       }
   }
   
   dependencies {
       implementation gradleApi()
       implementation localGroovy()
   
       // Android DSL  Android编译的大部分gradle源码
       implementation 'com.android.tools.build:gradle:3.4.0'
   }
   ```

   2. 修改相关的 build.gradle 文件，添加依赖，在根项目的 build.gradle 中添加：

      ![image-20220226171506534](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220226171508.png)

   3. 最后是构建在 gradle task 里面，运行 uploadArchives 任务即可，或者通过` ./gradlew uploadArchivers` 来执行这个 task：

   ![image-20220226171657830](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220226171659.png)

## 二、玩转Transform

Google官方在Android GradleV1.5.0版本以后提供了Transform API，它允许第三方插件在编译后的类文件转换为dex文件之前做处理操作，我们需要做的就是实现Transform来对.class文件便遍历来拿到所有方法，修改完成后再对源文件进行替换就可以了。感兴趣可以去看看[Transform版本历史](https://links.jianshu.com/go?to=http%3A%2F%2Fgoogle.github.io%2Fandroid-gradle-dsl%2Fjavadoc%2F)。

### 2.1 Transform的使用

在前面我们已经看到了怎样对一个transform进行注册，也就是我们在我们自定义的plugin中，通过如下进行注册，这里我选择使用kotlin来实现：

```kotlin
class HelloPlugin: Plugin<Project> {
    override fun apply(target: Project) {
        target.extensions.findByType(AppExtension::class.java)?.run {
            registerTransform(HelloTransform(target))
        }
    }
}
```

自定义的Transform是要继承于`com.android.build.api.transform.Transform`，可以看下[Transform文档介绍](https://google.github.io/android-gradle-dsl/javadoc/1.5/com/android/build/api/transform/Transform.html#getSecondaryFileOutputs--)，现在我们先来定义一个自定义的Transform（`不支持增量`）：

```kotlin
class HelloTransform: Transform() {
    /**
     * 返回对应的 Task 名称。
     */
    override fun getName(): String = "HelloTransform"

    /**
     * 输入文件的类型
     * 可供我们去处理的有两种类型, 分别是编译后的java代码, 以及资源文件(非res下文件, 而是assests内的资源)
     */
    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> = TransformManager.CONTENT_CLASS

    /**
     * 是否支持增量
     * 如果支持增量执行, 则变化输入内容可能包含 修改/删除/添加 文件的列表
     */
    override fun isIncremental(): Boolean = false

    /**
     * 指定插件的适用范围。
     */
    override fun getScopes(): MutableSet<in QualifiedContent.Scope> = TransformManager.SCOPE_FULL_PROJECT

    /**
     * transform的执行主函数
     * 进行具体的转换过程
     */
    override fun transform(transformInvocation: TransformInvocation?) {
      transformInvocation?.inputs?.forEach {
          // 输入源为文件夹类型   (本地 project 编译成的多个 class ⽂件存放的目录）
          it.directoryInputs.forEach {directoryInput->
              with(directoryInput){
                  // 获取class文件输出路径
                  val dest = transformInvocation.outputProvider.getContentLocation(
                      name,
                      contentTypes,
                      scopes,
                      Format.DIRECTORY
                  )
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了  
                  file.copyTo(dest)
              }
          }

          // 输入源为jar包类型（各个依赖所编译成的 jar 文件） 
          it.jarInputs.forEach { jarInput->
              with(jarInput){
                  // 获取jar包的输出路径
                  val dest = transformInvocation.outputProvider.getContentLocation(
                      name,
                      contentTypes,
                      scopes,
                      Format.JAR
                  )
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了  
                  file.copyTo(dest)
              }
          }
      }
    }
}
```

####  2.1.1 getName()

这个方法返回的就是我们的Transform名称，也就是会在Build的流程里会出现的：

![image-20220226184508782](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220226184510.png)

**这名字最终是如何构成的？**可以在gradle源码里有个`TransformManager`的类，这个类负责管理所有的Transform的子类，可以找到一个`getTaskNamePrefix`方法。会以tansform开头，之后拼接contentType，这个也就是Transform的输入类型，有`Classes`和`Resources`两种类型，最后就是会跟上我们这个`Transform`的`Name`了。

```java
#TransformManager

   static String getTaskNamePrefix(@NonNull Transform transform) {
        StringBuilder sb = new StringBuilder(100);
        sb.append("transform");

        sb.append(
                transform
                        .getInputTypes()
                        .stream()
                        .map(
                                inputType ->
                                        CaseFormat.UPPER_UNDERSCORE.to(
                                                CaseFormat.UPPER_CAMEL, inputType.name()))
                        .sorted() // Keep the order stable.
                        .collect(Collectors.joining("And")));
        sb.append("With");
        StringHelper.appendCapitalized(sb, transform.getName());
        sb.append("For");

        return sb.toString();
    }
```

#### 2.1.2 getInputTypes()

这个则是用来限定transform处理文件的类型，在对class文件进行处理时，返回的是TransformManager.CONTENT_CLASS`，而在对资源文件处理时，返回的是`TransformManager.CONTENT_RESOURCES`。除了CLASSES和RESOURCES，还有一些我们开发过程无法使用的类型，比如DEX文件，这些隐藏类型在一个独立的枚举类[ExtendedContentType](https://link.juejin.cn/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fstudio-master-dev%2Fbuild-system%2Fgradle-core%2Fsrc%2Fmain%2Fjava%2Fcom%2Fandroid%2Fbuild%2Fgradle%2Finternal%2Fpipeline%2FExtendedContentType.java%3Fautodive%3D0%2F%2F%2F)中，这些类型只能给Android编译器使用。

#### 2.1.3 getScopes()

这个是指定需要处理哪种文件，也就是用来表明作用域。可以看下有哪些选项：

```java
/**
 * 表示 Transform 要操作的内容范围，目前 Scope 有五种基本类型：
 *      1、PROJECT                   只有项目内容
 *      2、SUB_PROJECTS              只有子项目
 *      3、EXTERNAL_LIBRARIES        只有外部库
 *      4、TESTED_CODE               由当前变体（包括依赖项）所测试的代码
 *      5、PROVIDED_ONLY             只提供本地或远程依赖项
 *      SCOPE_FULL_PROJECT 是一个 Scope 集合，包含 Scope.PROJECT,
 */  
enum Scope implements ScopeType {
        /** Only the project (module) content */
        PROJECT(0x01),
        /** Only the sub-projects (other modules) */
        SUB_PROJECTS(0x04),
        /** Only the external libraries */
        EXTERNAL_LIBRARIES(0x10),
        /** Code that is being tested by the current variant, including dependencies */
        TESTED_CODE(0x20),
        /** Local or remote dependencies that are provided-only */
        PROVIDED_ONLY(0x40),
        /**
         * Only the project's local dependencies (local jars)
         *
         * @deprecated local dependencies are now processed as {@link #EXTERNAL_LIBRARIES}
         */
        @Deprecated
        PROJECT_LOCAL_DEPS(0x02),
        /**
         * Only the sub-projects's local dependencies (local jars).
         *
         * @deprecated local dependencies are now processed as {@link #EXTERNAL_LIBRARIES}
         */
        @Deprecated
        SUB_PROJECTS_LOCAL_DEPS(0x08);

       ......
    }
```

一般来说如果要处理所有class字节码的话，一般使用`TransformManager.SCOPE_FULL_PROJECT`，也就是如下：

```java
public static final Set<Scope> SCOPE_FULL_PROJECT =
            Sets.immutableEnumSet(
                    Scope.PROJECT,
                    Scope.SUB_PROJECTS,
                    Scope.EXTERNAL_LIBRARIES);
```

#### 2.1.4  inIncremental()

表示是否支持增量编译，关闭时就会进行全量编译，并且会删除上一次的输出内容。当我们开启增量编译的时候，input就包含了changed/removed/added/notchanged四种状态：

1. `NOTCHANGED`: 当前文件没有改变，不需处理，甚至复制操作都不用
2. `ADDED、CHANGED`: 有修改文件，并输出给下一个任务
3. `REMOVED`: outputProvider获取路径对应的文件被移除

#### 2.1.5 transform()

在这个方法中里我们将每个jar包和class文件赋值到dest路径下，这个dest路径也就是下一个Transform的输入数据，在复制的过程中，我们就可以对jar包和class文件的字节码进行修改`(ASM在这里飘过~)`。

处理后的class/jar包可以到/build/intermediates/transforms/HelloTransform/下查看，你会看到所有jar包都是123456递增着来的。可以看下获取输出路径的方法：

```java
# IntermediateFolderUtils
  
public synchronized File getContentLocation(String name, Set<ContentType> types, Set<? super Scope> scopes, Format format) {
        Preconditions.checkNotNull(name);
        Preconditions.checkNotNull(types);
        Preconditions.checkNotNull(scopes);
        Preconditions.checkNotNull(format);
        Preconditions.checkState(!name.isEmpty());
        Preconditions.checkState(!types.isEmpty());
        Preconditions.checkState(!scopes.isEmpty());
        Iterator var5 = this.subStreams.iterator();

        SubStream subStream;
        do {
            if (!var5.hasNext()) {
              //这里可以看到它是按位置递增
                SubStream newSubStream = new SubStream(name, this.nextIndex++, scopes, types, format, true);
                this.subStreams.add(newSubStream);
                return new File(this.rootFolder, newSubStream.getFilename());
            }

            subStream = (SubStream)var5.next();
        } while(!name.equals(subStream.getName()) || !types.equals(subStream.getTypes()) || !scopes.equals(subStream.getScopes()) || format != subStream.getFormat());

        return new File(this.rootFolder, subStream.getFilename());
    }
```

### 2.2 Transform的原理

介绍了如何使用Transfoem之后，我们再来看下它的原理(gradle插件7.0`.2`版本)。

首先我们来看下从Java源代码到apk的过程，如下图：

![image-20220226221112974](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220226221114.png)

从这里我们可以清楚看到gradle的打包过程基本上是通过官方的`Transform`来完成。而每个`Transform`其实都是一个`gradle task`，Android编译器中的TaskManager将每个`Transform`串连起来，第一个`Transform`接收来自javac编译的结果，以及本地的第三方依赖还有asset目录下的`resource`资源。然后这些编译的中间产物会在`Transform`组成的链条上流动，每一个`Tansform`会对class进行处理之后再传给下一个`Transform`。

而我们自定的`Transform`会插入到这个`Tansform`链条的最前面，要优先于`ProguardTransform`执行的，所以不会造成因为混淆而无法扫描到类信息。

#### 2.2.1  TransformManager

在前面自定义plugin中调用`registerTransform`对`transform`进行注册时，实际上是放入了`BaseExtension`类中的list数组里，然后是由`TaskManager`调用了`TransformManager的`addTransform`方法。这里`TransformManager`管理了项目对应变体的所有`Transform`对象。

我们来看下addTransform方法具体实现：

```java
    public <T extends Transform> Optional<TaskProvider<TransformTask>> addTransform(
            @NonNull TaskFactory taskFactory,
            @NonNull TransformVariantScope scope,
            @NonNull T transform,
            @Nullable PreConfigAction preConfigAction,
            @Nullable TaskConfigAction<TransformTask> configAction,
            @Nullable TaskProviderCallback<TransformTask> providerCallback) {

       ......
         
        List<TransformStream> inputStreams = Lists.newArrayList();
        //transform task的命名
        String taskName = scope.getTaskName(getTaskNamePrefix(transform));

        // 获取仅引用型流
        List<TransformStream> referencedStreams = grabReferencedStreams(transform);

        // 找到输入流, 并计算通过transform的输出流
        IntermediateStream outputStream = findTransformStreams(
                transform,
                scope,
                inputStreams,
                taskName,
                scope.getGlobalScope().getBuildDir());

      ......

        transforms.add(transform);

        // transform task的创建
        return Optional.of(
                taskFactory.register(
                        new TransformTask.CreationAction<>(
                                scope.getFullVariantName(),
                                taskName,
                                transform,
                                inputStreams,
                                referencedStreams,
                                outputStream,
                                recorder),
                        preConfigAction,
                        configAction,
                        providerCallback));
    }
```

1.   在`getTaskNamePrefix`方法中会定义task的名字，前面也已经分析过了。

2. 然后在`grabReferencedStreams`方法中，对transform的数据输入，通过内部定义的引用型输入的Scope和ContentType两个维度进行过滤，可以看到`grabReferencedStreams`方法里求取与`streams`作用域和作用类型的交集来获取对应的流, 将其定义为我们需要的引用型流。

   ```java
   private List<TransformStream> grabReferencedStreams(@NonNull Transform transform) {
           Set<? super Scope> requestedScopes = transform.getReferencedScopes();
           ......
             
           List<TransformStream> streamMatches = Lists.newArrayListWithExpectedSize(streams.size());
   
           Set<ContentType> requestedTypes = transform.getInputTypes();
           for (TransformStream stream : streams) {
               Set<ContentType> availableTypes = stream.getContentTypes();
               Set<? super Scope> availableScopes = stream.getScopes();
   
               Set<ContentType> commonTypes = Sets.intersection(requestedTypes,
                       availableTypes);
               Set<? super Scope> commonScopes = Sets.intersection(requestedScopes, availableScopes);
   
               if (!commonTypes.isEmpty() && !commonScopes.isEmpty()) {
                   streamMatches.add(stream);
               }
           }
   
           return streamMatches;
       }
   ```

3. 之后在`findTransformStreams`方法中，会根据定义的SCOPE和INPUT_TYPE，获取对应的消费型输入流，移除这一部分的消费性的输入流。为所有类型和范围创建单个组合输出流，并将其添加到下一次转换的可用流列表中。

   ```java
   private IntermediateStream findTransformStreams(
               @NonNull Transform transform,
               @NonNull TransformVariantScope scope,
               @NonNull List<TransformStream> inputStreams,
               @NonNull String taskName,
               @NonNull File buildDir) {
   
           ......
           Set<ContentType> requestedTypes = transform.getInputTypes();
     		//在streams中移除对应的消费型输入流
           consumeStreams(requestedScopes, requestedTypes, inputStreams);
   
           // 创建输出流
           Set<ContentType> outputTypes = transform.getOutputTypes();
   
           File outRootFolder =
                   FileUtils.join(
                           buildDir,
                           StringHelper.toStrings(
                                   AndroidProject.FD_INTERMEDIATES,
                                   FD_TRANSFORMS,
                                   transform.getName(),
                                   scope.getDirectorySegments()));
   
           // 输出流的创建
           IntermediateStream outputStream =
                   IntermediateStream.builder(
                                   project,
                                   transform.getName() + "-" + scope.getFullVariantName(),
                                   taskName)
                           .addContentTypes(outputTypes)
                           .addScopes(requestedScopes)
                           .setRootLocation(outRootFolder)
                           .build();
           streams.add(outputStream);
           return outputStream;
       }
   ```

4. 最后将新创建的`TransformTask`注册到`TaskManager`中。

#### 2.2.2 TransformTask

在这个类里我们看到最终`transform`的`tansform`方法被调用，也就是在其对应的TaskAction方法中执行：

```java
# TransformTask
  
    @TaskAction
  void transform(final IncrementalTaskInputs incrementalTaskInputs)
            throws IOException, TransformException, InterruptedException {

        final ReferenceHolder<List<TransformInput>> consumedInputs = ReferenceHolder.empty();
        final ReferenceHolder<List<TransformInput>> referencedInputs = ReferenceHolder.empty();
        final ReferenceHolder<Boolean> isIncremental = ReferenceHolder.empty();
        final ReferenceHolder<Collection<SecondaryInput>> changedSecondaryInputs =
                ReferenceHolder.empty();

        isIncremental.setValue(transform.isIncremental() && incrementalTaskInputs.isIncremental());

  //在增量模式下, 判断输入流(引用型和消费型)的变化
      ......

        recorder.record(
                ExecutionType.TASK_TRANSFORM,
                executionInfo,
                getProject().getPath(),
                getVariantName(),
                new Recorder.Block<Void>() {
                    @Override
                    public Void call() throws Exception {

                        transform.transform(
                                new TransformInvocationBuilder(TransformTask.this)
                                        .addInputs(consumedInputs.getValue())
                                        .addReferencedInputs(referencedInputs.getValue())
                                        .addSecondaryInputs(changedSecondaryInputs.getValue())
                                        .addOutputProvider(
                                                outputStream != null
                                                        ? outputStream.asOutput(
                                                                isIncremental.getValue())
                                                        : null)
                                        .setIncrementalMode(isIncremental.getValue())
                                        .build());

                        if (outputStream != null) {
                            outputStream.save();
                        }
                        return null;
                    }
                });
    }
```

到这里我们已经知道了Transform的数据流动原理、输入的类型和过滤机制。

### 2.3 Transform的增量与并发

学习了上面之后我们可以轻松的定义出一个Transform。可是每次编译`transform`方法都会执行，就会遍历所有的class文件，会解压所有jar文件，然后重新压缩成所有的jar文件，这样就会拖慢编译的时间。如何解决，这里我们就用到了增量编译。

但不是每次的编译都可以增量编译，毕竟第一次编译或clean后重新编译`directory.changedFiles`为空，需要做好区分经测试。如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理。如果是增量编译，则要检查每个文件的Status，Status分为四种`NOTCHANGED`/`ADDED`/`CHANGED`/`REMOVED`，并且对四种文件的操作不尽相同。

可以来看下增量编译的代码实现，**详细代码可以看下-->[BaseTransform](https://github.com/LebronXia/TransformKtsDemo/blob/main/buildSrc/src/main/java/com/xiamu/transform/base/BaseTransform.kt)**

```kotlin
    //进行具体的转换过程。
    override fun transform(transformInvocation: TransformInvocation) {
        Log.log("transform start--------------->")
        onTransformStart()

        val outputProvider = transformInvocation.outputProvider
        val context = transformInvocation.context
        val isIncremental = transformInvocation.isIncremental
        val startTime = System.currentTimeMillis()

        //不是增量更新，删除之前输出
        if (!isIncremental){
            outputProvider.deleteAll()
        }

        transformInvocation?.inputs?.forEach{input ->
            //输入为文件夹类型   (本地 project 编译成的多个 class ⽂件存放的目录）
            input.directoryInputs.forEach{directoryInput ->
                submitTask {
                    handleDirectory(directoryInput, outputProvider, context, isIncremental)
                }
            }

            //输入为jar包类型  （各个依赖所编译成的 jar 文件）
            input.jarInputs.forEach{jarInput ->
                submitTask {
                    handleJar(jarInput, outputProvider, context, isIncremental)
                }
            }
        }

        val taskListFeature = executorService.invokeAll(taskList)
        taskListFeature.forEach{
            it.get()
        }
        onTransformEnd()
        Log.log("transform end--------------->" + "duration : " + (System.currentTimeMillis() - startTime) + " ms")
    }
```

对输入为jar包类型处理：

```kotlin
    private fun handleJar(jarInput: JarInput, outputProvider: TransformOutputProvider, context: Context, isIncremental: Boolean) {
        //得到上一个Transform输入文件
        val inputJar = jarInput.file
        //得到当前Transform输出jar文件
        val outputJar = outputProvider.getContentLocation(
            jarInput.name, jarInput.contentTypes,
            jarInput.scopes, Format.JAR
        )

        //增量处理
        if (isIncremental){
            when(jarInput.status){
                //文件没有改变
                Status.NOTCHANGED -> {
                }
                //有修改文件
                Status.ADDED,Status.CHANGED -> {
                }
                //文件被移除
                Status.REMOVED -> {
                    if (outputJar.exists()){
                        FileUtils.forceDelete(outputJar)
                    }
                    return
                }
                else -> {
                    return
                }
            }
        }

        if (outputJar.exists()){
            FileUtils.forceDelete(outputJar)
        }

        //修改后的文件
        val modifiedJar = if (ClassUtils.isLegalJar(jarInput.file)) {
            modifyJar(jarInput.file, context.temporaryDir)
        } else {
            Log.log("不处理： " + jarInput.file.absoluteFile)
            jarInput.file
        }
        FileUtils.copyFile(modifiedJar, outputJar)
    }
```
对输入为文件夹类型处理：

```kotlin
    private fun handleDirectory(directoryInput: DirectoryInput, outputProvider: TransformOutputProvider, context: Context, isIncremental: Boolean) {

        //得到上一个Transform输入文件目录
        val inputDir = directoryInput.file
        //得到当前Transform
        val outputDir = outputProvider.getContentLocation(
            directoryInput.name, directoryInput.contentTypes,
            directoryInput.scopes, Format.DIRECTORY
        )

        val srcDirPath = inputDir.absolutePath
        val destDirPath = outputDir.absolutePath
        //写入临时文件的目录
        val temporaryDir = context.temporaryDir

        //创建目录
        FileUtils.forceMkdir(outputDir)
        if (isIncremental){
            directoryInput.changedFiles.entries.forEach { entry ->
                val inputFile = entry.key
                //最终文件应该存放的路径
//                val destFilePath = inputFile.absolutePath.replace(srcDirPath, destDirPath)
//                val destFile = File(destFilePath)
                when(entry.value){
                    Status.ADDED, Status.CHANGED ->{
                        //处理class文件
                        modifyClassFile(inputFile, srcDirPath, destDirPath, temporaryDir)
                    }
                    Status.REMOVED -> {
                        val destFilePath = inputFile.absolutePath.replace(srcDirPath, destDirPath)
                        val destFile = File(destFilePath)
                        if (destFile.exists()){
                            destFile.delete()
                        }
                    }
                    Status.NOTCHANGED -> {
                        
                    }
                }
            }
        } else {
            //过滤出是文件的，而不是目录
            directoryInput.file.walkTopDown().filter { it.isFile }
                .forEach { classFile ->
                    modifyClassFile(classFile, srcDirPath, destDirPath, temporaryDir)
                }
        }
    }
```

根据这个就能为我们的编译插件提供增量的特性。

## 三、结语

通过本文，我们学习了如何自定义一个Gradle插件，如何定义一个`Transform`以及`Transform`的内部原理。学完这些还是不够的，跟之后要讲到的ASM结合起来，你就能利用字节码插桩技术为所欲为了。


参考：

[深度探索 Gradle 自动化构建技术（四、自定义 Gradle 插件）](https://juejin.cn/post/6844904135314128903)

[写个更牛逼的Transform | Plugin 进阶教程](https://juejin.cn/post/6916304559602139149)

[Android Transform增量编译](https://juejin.cn/post/6844904150925312008)

[Gradle+Transform+Asm自动化注入代码](https://www.jianshu.com/p/fffb81688dc5)

[Transform API](https://links.jianshu.com/go?to=http%3A%2F%2Fgoogle.github.io%2Fandroid-gradle-dsl%2Fjavadoc%2F)

[如何开发一款高性能的gradle transform](https://www.jianshu.com/p/d84032b46b56)

[一起玩转Android项目中的字节码](https://juejin.cn/post/6844903734066036744)

[深入理解Transform](https://juejin.cn/post/6844903829671002126)

[Transform详解](https://www.jianshu.com/p/37a5e058830a)

[性能优化之Web View预加载](https://www.jianshu.com/p/ff042054f8dc)
[ASMInjectDemo](https://github.com/changer0/ASMInjectDemo)

[AndroidStudio发布项目到Maven(JCenter)](https://www.jianshu.com/p/bdcd00c2470b)

