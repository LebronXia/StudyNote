Android Studio默认的工具是Gradle，通常开发者不需要了解Gradle的脚本配置，也能开发出一个App，但是如果你需要修改打包中的输出目录、提高打包速度的话，就要对Gradle有个深入的了解了。而在开发学习Gradle不能仅仅把它当做一个工具来学习，更应该把它当成编程框架来看，这里是[Gradle API文档](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)，我们再编写编译脚本，其实就是在玩 Gradle的API。

Gradle的组成可以分为三部分：

1. groovy核心语法
2. Build script block
3. Gradle API：包含project、Task/Settiing等等，也是接下来重点介绍的。

## Gradle优势

1. 在灵活性上，Gradle支持基于[groovy](https://so.csdn.net/so/search?q=groovy)语言编写脚本，侧重于构建过程的灵活性，适合于构建复杂度较高的项目，可以完成非常复杂的构建。
2. 在粒度性上，Gradle 构建的粒度细化到了每一个 task 之中。并且它所有的 Task 源码都是开源的，在我们掌握了这一整套打包流程后，我们就可以通过去修改它的 Task 去动态改变其执行流程。
3. 在扩展性上，Gradle 支持插件机制，所以我们可以复用这些插件，就如同复用库一样简单方便。

## Grale 构建生命周期

下面来看下gradle的执行命令，执行`./gradlew build  `  可以说是最复杂的task命令了。 (./gradlew是mac中的语法)

![image-20220108201559635](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220109191437.png)可以看到每次都会执行很长时间，是因为build的task任务依赖了很多任务，它会把它所依赖的task执行完之后，才会执行自己的task.。我们还是先了解下Gradle的构建过程。

Gradle的构建过程可以分为三部分：**初始化阶段**、**配置阶段**和**执行阶段**。

![image-20220108234033023](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220109191447.png)

### 初始化阶段

Gradle为每个项目创建一个Project实例，在多项目构建中，Gradle会找出哪些项目依赖需要参与到构建中。与初始化相关的的脚本是`settings.gradle`，会读出整个工程中有多少个project.

### 配置阶段

配置阶段的任务是执行各项目下的build.gradle脚本，完成Project配置，并且构造Task任务依赖关系图以便在执行阶段按照依赖关系执行Task。

每个`build.gradle`对应一个`Project`对象，配置阶段执行的代码包括`build.gralde`中的各种语句、闭包以及`Task`中的配置段语句。

### 执行阶段

在配置阶段结束后，Gradle会根据任务Task的依赖关系会创建一个有向无环图，还可以通通过Gradle对象的getTaskGraph方法访问，对应的类是TaskExecutionGraph，然后通过调用gradle <任务>执行对应任务。

### Gradle生命周期监听

#### Project中添加生命周期监听

现在我们来看下平时开发中最常用的监听方法：

```groovy
//在 Project 进行配置前调用
void beforeEvaluate(Closure closure)
//在 Project 配置结束后调用
void afterEvaluate(Closure closure)
```

接下来用个例子来验证下结果：

setting.gradle

```groovy
include ':app'

println '初始化阶段开始...'
```

根目录下的build.gradle

```groovy
//对子模块进行配置
subprojects { sub ->
    sub.beforeEvaluate { proj ->
        println "子项目beforeEvaluate回调..."
    }
}

println "根项目配置开始---"

task rootTest {
    println "根项目里任务配置---"
    doLast {
        println "执行根项目任务..."
    }
}

println "根项目配置结束---"
```

app目录下的build.gradle

```groovy
println "APP子项目配置开始---"

afterEvaluate {
    println "APP子项目afterEvaluate回调..."
}

task appTest {
    println "APP子项目里任务配置---"
    doLast {
        println "执行子项目任务..."
    }
}

println "APP子项目配置结束---"
```

执行下`./gradlew clean,得到如下的打印结果：

```groovy
初始化阶段开始...
根项目配置开始---
根项目里任务配置---
根项目配置结束---
子项目beforeEvaluate回调...
APP子项目配置开始---
APP子项目里任务配置---
APP子项目配置结束---
APP子项目afterEvaluate回调...
```

  #### Gradle中添加生命周期监听

```groovy
include ':app'
gradle.addBuildListener(new BuildListener() {
  
  //构建开始前调用
    void buildStarted(Gradle var1) {
        println '构建开始...'
    }
  
  //settings.gradle配置完后调用，只对settings.gradle设置生效
    void settingsEvaluated(Settings var1) {
        // var1.gradle.rootProject 这里访问 Project 对象时会报错，
        // 因为还未完成 Project 的初始化。
        println 'settings：执行settingsEvaluated...'
    }
  
  //当settings.gradle中引入的所有project都被创建好后调用，只在该文件设置才会生效
    void projectsLoaded(Gradle var1) {
        println 'settings：执行projectsLoaded...'
        println '初始化结束，可访问根项目：' + var1.gradle.rootProject
    }
  
  //所有project配置完成后调用
    void projectsEvaluated(Gradle var1) {
        println 'settings: 执行projectsEvaluated...'
    }
  
  //在project进行配置前调用，child project必须在root project中设置才会生效，root project必须在settings.gradle中设置才会生效
  	void beforeProject { proj ->
    	println "settings：执行${proj.name} beforeProject"
		}

  //在project配置后调用
		void afterProject { proj ->
    	println "settings：执行${proj.name} afterProject"
		}
  
  //构建结束后调用
    void buildFinished(BuildResult var1) {
        println '构建结束 '
    }
})
```

看下输出结果：

```groovy
settings：执行settingsEvaluated...
settings：执行projectsLoaded...
settings：执行test beforeProject
根项目配置开始---
根项目里任务配置---
根项目配置结束---
settings：执行test afterProject
settings：执行app beforeProject
子项目beforeEvaluate回调...
APP子项目配置开始---
APP子项目里任务配置---
APP子项目配置结束---
settings：执行app afterProject
APP子项目afterEvaluate回调...
settings: 执行projectsEvaluated...
构建结束...
```

另外还有其他监听方法：

```groovy
this.gradle.beforeProject{}  //等同于beforeEvaluate
this.graddle.afterProject{}  //等同于afterEvaluate

//下面几种都是可以监听的
this.gradle.addListener
tthis.gradle.addProjectEvaluationListener
```

在了解了gradle的生命周期监听后，还有个疑问为什么执行build命令的时候，会其他命令的输出。下面我们来看一张图：

![image-20220109130350195](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220109191457.png)

这张图会在配置阶段完成，形成一个有向无环图，在执行名字为build的task的时候，所以会有其他依赖的task都要执行完。

### TaskExecutionGraph API介绍

[TaskExecutionGraph API文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2Fexecution%2FTaskExecutionGraph.html)

| API                            | 描述                       |
| ------------------------------ | -------------------------- |
| addTaskExecutionGraphListener  | 给任务执行图添加监听       |
| addTaskExecutionListener       | 给任务的执行添加监听       |
| whenReady                      | 当任务执行图填充完毕被调用 |
| beforeTask                     | 当一个任务执行之前被调用   |
| afterTask                      | 当一个任务执行完毕被调用   |
| hasTask                        | 查询是否有这个task         |
| getAllTasks                    | 获取所有的Task             |
| Set getDependencies(Task task) | 返回参数Task的依赖关系     |

## Project详解

每一个build.gradle脚本文件被Gradle加载解析后，都是会生成一个Project对象。在脚本中的配置方法其实都对应着Project中的API。 

[Project API 文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2FProject.html)

在开始前，我们先介绍一些最常用到的API,具体的可以看下文档：

| API                                              | 描述                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| getRootProject()                                 | 获取根Project                                                |
| getRootDir                                       | 返回根目录文件夹                                             |
| getBuildDir                                      | 返回构建目录，所有的build生成物都放入到这个里面              |
| setBuildDir(File path)                           | 设置构建文件夹                                               |
| getParent()                                      | 获取此Project的父Project                                     |
| getChildProjects                                 | 获取此Project的直系子Project                                 |
| setProperty(String name, @Nullable Object value) | 给此Project设置属性                                          |
| getProject()                                     | 但会当前Project对象，可用于访问当前Project的属性和方法       |
| getAllprojects                                   | 返回包括当前Project，以及子Project的集合                     |
| allprojects(Closure configureClosure)            | 返回包括当前Project，以及子Project的集合到闭包中             |
| getSubprojects                                   | 返回当前Project下的所有子Project                             |
| subprojects(Closure configureClosure)            | 返回当前Project下的所有子Project到闭包中                     |
| Task task(String name)                           | 创建一个Task，添加到此Priject                                |
| getAllTasks(boolean recursive)                   | 如果recursive为true那么返回当前Project和子Project的全部Task，如果为false只返回当前Project的所有task |
| getTasksByName(String name, boolean recursive)   | 根据名字返回Task，如果recursive为true那么返回当前Project和子Project的Task，如果为false只返回当前Project的task |
| beforeEvaluate(Closure closure)                  | 在Project评估之前调用                                        |
| afterEvaluate(Closure closure);                  | 在项目评估之后调用                                           |
| hasProperty(String propertyName)                 | 查看是否存在此属性                                           |
| getProperties()                                  | 获取所有属性                                                 |
| findProperty(String propertyName);               | 返回属性的value                                              |
| dependencies(Closure configureClosure)           | 为Project配置依赖项                                          |
| buildscript(Closure configureClosure)            | 为Project配置build脚本                                       |
| project(String path, Closure configureClosure)   | 根据路径获取Project实例，并在闭包中配置Project               |
| getTasks()                                       | 返回此Project中所有的tasks                                   |

首先我们来看下一个项目中有多少个Project?可以来执行`./gradlew projects`命令。

可以看到输出的结果：

```groovy
------------------------------------------------------------
Root project
------------------------------------------------------------

Root project 'GradleTestDemo'
\--- Project ':app'
```

>  project方法都是执行在配置阶段

接下来我们通过一些实际的例子，由浅如深的来体会这些API的含义。下面所有的例子都是在根目录下的`build.gradle`执行。

### Project API分解

#### 1. getAllprojects

```groovy
this.getProjects()

def getProjects(){
    println '------'
    println 'Root project'
    println '-----'
    getAllprojects().eachWithIndex{ Project project, int index ->
      //下标为 0，表明当前遍历的是 rootProject
        if (index == 0){
            println "Root prohrct ':${project.name}"
        } else {
            println "+---- project ':${project.name}"
        }
    }
}
```

直接执行./`gradlew clean` 命令，可以看到在配置阶段输出以下结果：

```groovy
------
Root project
-----
Root prohrct ':GradleTestDemo
+---- project ':app

```

#### 2.getSubprojects

表示获取当前工程下所有子工程的实例

```groovy
this.getSubProjects()

def getSubProjects() {
    println "<================>"
    println " Sub Project Start "
    println "<================>"
    // getSubprojects 方法返回一个包含子 project 的 Set 集合
    this.getSubprojects().each { Project project ->
        println "child Project is $project"
    }
}
```

#### 3.getParent

getParent 表示 **获取当前 project 的父类，需要注意的是，如果我们在根工程中使用它，获取的父类会为 null，因为根工程没有父类。

#### 4.getRootProject

如果我们想在根工程仅仅获取当前的 project 实例该怎么办呢？**直接使用 getRootProject 即可在任意 build.gradle 文件获取当前根工程的 project 实例。

```groovy
this.getRootPro()

def getRootPro() {
    def rootProjectName = this.getRootProject().name
    println "root project is $rootProjectName"
}
```

#### 5. project

指定app进行配置, 在父工程中完成子工程中的配置

```groovy
project('app'){
    Project project ->
        apply plugin: 'com.android.application'
        group 'com.haha'
        version '1.0.0-release'
        dependencies {
        }
        android{
        }
}
```

配置当前节点工程和其subproject的所有工程

```groovy
allprojects {
		group 'com.haha'
    version '1.0.0-release'
}
```

不包括当前结点工程，只包括它的子subproject

```groovy
subprojects { Project project ->
    if (project.plugins.hasPlugin('com.android.library')){
        apply from: '../publishToMaven.gradle'
    }
}
```

### 属性相关API分解

默认定义属性：

```groovy
public interface Project extends Comparable<Project>, ExtensionAware, PluginAware {
  //默认从build.gradle中读取配置
    String DEFAULT_BUILD_FILE = "build.gradle";
  //路径分隔符，用“:”代表分隔符
    String PATH_SEPARATOR = ":";
  //代表一个输出文件夹
    String DEFAULT_BUILD_DIR_NAME = "build";
    String GRADLE_PROPERTIES = "gradle.properties";
    String SYSTEM_PROP_PREFIX = "systemProp";
    String DEFAULT_VERSION = "unspecified";
 		String DEFAULT_STATUS = "release";
  	......
}
```

其中定义的属性可能你会觉得太少，但gradle也支持扩展的属性。而定义扩展属性总共有两种方式：

**第一种**最常见的做法是在根目录下通过 ext 命名空间来定义属性：

```groovy
// 在根目录下的 build.gradle 中
ext {
    compileSdkVersion = 27
    buildToolsVersion = "28.0.3"
}

//子build.gradle直接调用父类扩展属性，根project下所有的属性会再子project中被继承
compileSdkVersion this.compileSdkVersion
```

在项目越来越庞大的时候，仍然定义在根build.gradle下，会让根目录显得越来越复杂。这时候我们可以开启一个新的gradle脚本专门来定义扩展属性。将其命名为 common.gradle，如下所示：

```groovy
//用来存放应用中的所有配置变量，统一管理
ext {
    android = [
            compileSdkVersion   : 30,
            versionCode         : 1,
            versionName         : '1.0.0',
    ]
}
```

在根build.gradle文件下来引入，会从当前工程下寻找common.gradle.

```groovy
apply from: this.file('common.gradle')
```

在子build.gradle中就可以调用到我们的扩展属性了。

```groovy
compileSdkVersion this.rootProject.ext.android.compileSdkVersion
```

**第二种**是在`gradle.properties`中定义一个变量

```groovy
isLoadTest = false
```

然后再`settiing.gradle`中进行处理

```groovy
if(hasProperty('isLoadTest') ? isLoadTest.toBoolean() : false){
  include ':Test'
}
```

### 文件属性操作分解

#### 1. 路径获取

```groovy
//根工程文件路径
println getRootDir().absolutePath
//build文件地址
println getBuildDir().absolutePath
//当前文件根目录
println getProjectDir().absolutePath
```

#### 2.通过file/files定位文件

```groovy
//定位单个文件
println getContent("gradle.properties")
def getContent(String path){
    try {
        //相对于当前的project工程开始查找
        def file = file(path)
        return file.text
    } catch(GradleException e){
        println 'file not found'
    }
    return null
}

//文件集合，Gradle里用 FileCollection 来表示
FileCollection fileCollection = files("${buildDir}/test", "${buildDir}/test2")
println "-------对文件集合进行迭代--------"
fileCollection.each {File f ->
    println f.name
}
println "-------文件迭代结束-------"

//获取文件列表
Set<File> set = fileCollection.getFiles()
println "文件集合里共有${set.size()}个文件"
```

#### 4.通过copy拷贝文件

```groovy
//文件拷贝
copy {
    print 'the file'
    from file ('build/outputs/apk/')
    into getRootDir().path + '/apk66/'
    //进行文件排除
    exclude{
    }
    //重命名文件
    rename('app-debug.apk', 'haha.apk')

}
```

#### 5.对文件数进行遍历

我们可以 **使用 fileTree 将当前目录转换为文件数的形式，然后便可以获取到每一个树元素（节点）进行相应的操作**。

```groovy
//对文件树进行遍历
//将apk下的每个文件拷贝到test目录下
fileTree('build/outputs/apk'){FileTree fileTree ->
    fileTree.visit{ FileTreeElement element ->
        print 'the file name is:' + element.file.name
        copy {
            from element.file
            into getRootProject().getBuildDir().path + '/test/'
        }
    }
}
```

### 依赖相关API分解

```groovy
buildscript {
    // 配置我们工程的仓库地址
    repositories {
        google()
        jcenter()
        mavenCentral()
        maven { url 'https://maven.google.com' }
        maven { url "https://plugins.gradle.org/m2/" }
        maven {
            url uri('../PAGradlePlugin/repo')
        }
    }
    
    // 配置我们工程的插件依赖
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.4'
        
        ...
    }
```

在app module目录下的`dependendencies`是来为应用程序添加第三方依赖的，这里我们需要注意下exclude和transitives使用的用法。

```groovy
implementation(rootProject.ext.dependencies.glide) {
        // 排除依赖：一般用于解决资源、代码冲突相关的问题
        exclude module: 'support-v4' 
  			exclude group: 'com.androoid.support'
        // 传递依赖：A => B => C ，B 中使用到了 C 中的依赖，
        // 且 A 依赖于 B，如果打开传递依赖，则 A 能使用到 B 
        // 中所使用的 C 中的依赖，默认都是不打开，即 false
        transitive false 
}
```

![image-20220109183453918](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220109183459.png)

一般工程A是不允许直接调用工程C下，所以一般为false。

> 提一下`provided`引入只在编译中起作用，不在运行中起作用。解决利用占位编译，防止重复引用。

### gradle执行外部命令

我们一般是 **使用 Gradle 提供的 exec 来执行外部命令**

```groovy
//执行外部命令
task apkcopy(group: 'imooc'){
    doLast {
      //gradle执行阶段去执行
        def sourcePath = this.buildDir.path + '/outputs/apk'
        def desationPath = '/Users/zhengxiaobo/Downloads/'
        def command = "mv -f ${ sourcePath} ${desationPath}"
        //执行外部命令
        exec {
            try {
                executable 'bash'   //执行类型
                args '-c', command    //唯一会变的就是commond,在执行其他的gradle命令时，修改这里就可以了
                println 'the coommand is execute success'
            } catch(GradleException e){
                println 'the command is execute failed'
            }
        }

    }
}
```

## Task详解

一个 Task 是 Gradle 里项目构建的原子执行单元，Gradle 通过将一个个Task串联起来完成具体的构建任务，每个 Task 都属于一个 Project。

[Task API文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fdsl%2Forg.gradle.api.Task.html)

开始前，我们执行`./gradlew tasks`来查看项目中有多少Task。

可以看到如下输出：

![image-20220110100252043](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220110100304.png)

上图就是当前工程中的每条task都已罗列出，并且有黄色的输出表示当前`task`的描述。

### Task定义及配置

常见的有两种定义方式：

```groovy
//直接通过task函数创建
task helloTask{
  println "i am hellloTask"
}

//通过TaskContationer去创建Task
this.tasks.creat(name: "helloTask2"){
  println "i am helloTasks2"
}
```

>  TaskContianer 是用来管理所有的 Task 实例集合的，可以通过 Project.getTasks() 来获取 TaskContainer 实例。

不管哪种创建Task的同时，我们还可以配置它的一些列属性，如：

```groovy
//定义分组 添加注释
task helloTask(group: 'test', description: 'task study'){
  println 'i am hellpTask'
}

this.tasks.create(name: 'hellpTask2'){
  setGroup('imooc')
  setDescription('task study')
  println 'i am hellpTask2'
}
```

添加了group后，我们打开侧边Tasks目录下就会多出个test的分组，更方便查找。

目前 官方所支持的属性 可以总结为如下表格：

| 选型              | 描述                                                      | 默认值       |
| ----------------- | --------------------------------------------------------- | ------------ |
| "name"            | task 名字                                                 | 无，必须指定 |
| "type"            | 需要创建的 task Class                                     | DefaultTask  |
| "action"          | 当 task 执行的时候，需要执行的闭包 closure 或 行为 Action | null         |
| "overwrite"       | 替换一个已存在的 task                                     | false        |
| "dependsOn"       | 该 task 所依赖的 task 集合                                | []           |
| "group"           | 该 task 所属组                                            | null         |
| "description"     | task 的描述信息                                           | null         |
| "constructorArgs" | 传递到 task Class 构造器中的参数                          | null         |

在开发中，如何引用另一个task的属性？如下所示：

```groovy
//使用 defaultTasks 关键字 来将一些任务标识为默认的执行任务
defaultTasks "Gradle_First", "Gradle_Last"

task Gradle_First() {
  //使用 ext 给 task 自定义需要的属性
    ext.good = true
}

task Gradle_Last() {
    doFirst {
        println Gradle_First.goodg
    }
    doLast {
      //使用 "$" 来引用另一个 task 的属性
        println "I am not $Gradle_First.name"
    }
}
```

### Task执行详解

在gradle脚本的配置阶段都会执行，也就是说不管执行脚本里的哪个任务，所有的task里的配置代码都会被执行，但是我们期望的是调用一个方法的时候，这个方法里的代码才会执行。这里就要涉及到Task Action的概念了。

Task通常用`doFirs`t和`dolast`两个方式用于在执行期间进行操作：

```groovy
task helloTask(group: 'test', description: 'task study'){
  println 'i am hellpTask'
  //表示 task 执行最开始的时候被调用的 Action。
  doFirst{
    println 'the task group is：' + group
  }
  
  //表示 task 将执行完的时候被调用的 Action。
  doLast{
    println 'the task doLast'
  }
}
helloTask.doFirst{
   println 'the task description is：' + description
}
```

可以看到输出结果：

```groovy
the task description is：task study
the task group is：test
the task doLast
```

这样我们就知道了单独调用方法会先执行，然后配置块中的再执行。

接下来，就用一个实战更加了解`doFirst`和`doLast`的用法，来实现计算build执行期间的耗时：

```groovy
//计算build执行时长
def startBuildTime, endBuildTime

 //保证要找的task已经配置完毕
this.afterEvaluate { Project project ->
   // 执行build任务时，第一个被执行的Task
        def preBuildTask = project.tasks.getByName('preBuild')
        preBuildTask.doFirst {
          //获取第一个 task 开始执行时刻的时间戳
            startBuildTime = System.currentTimeMillis()
        }

				//找到当前 project 下最后一个执行的 task，即 build task
        def buildTask = project.tasks .getByName('build')
        buildTask.doLast {
						//获取最后一个 task 执行完成前一瞬间的时间戳
            endBuildTime = System.currentTimeMillis()
            println "Current project execute time is ${endBuildTime - startBuildTime}"
        }
}
```

### Task的依赖和执行顺序

在task执行顺序方式，有三种方式可以规定task的执行顺序：

#### 1.dependsOn强依赖方式

通过下面例子来了解Task如何进行依赖：

```groovy
task taskX{
    doLast{
        println 'taskX'
    }
}

task taskY{
    doLast{
        println 'taskY'
    }
}

//通过定义dependsOn依赖了taskX 和tasky
task taskZ(dependsOn: [taskX, taskY]){
  doLast{
    print 'taskZ'
  }
}
//另一种方式
taskZ.dependsOn(taskX, taskY)
```

再执行后就可以看到先执行了taskX和taskY，在执行taskZ。

```groovy
> Task :app:taskX
taskX

> Task :app:taskY
taskY

> Task :app:taskZ
taskZ
```

如果定义一个task不知道依赖具体的task的时候该怎么办？这时就要我们动态的进行依赖。

```groovy
task lib1 {
    doLast{
        println 'lib1'
    }
}

task taskZ(dependsOn: [taskX, taskY]){
    //依赖所有以lib开头的任务
    dependsOn this.tasks.findAll{task ->
        return task.name.startsWith('lib')
    }
    doLast{
        println 'taskY'
    }
}
```

下面还是用一个例子来介绍下task依赖在实战中的使用，来解析一个xml文件并进行输出：

```groovy
//解析xml文件
task handleReleaseFile{
    def srcFile = file('releases.xml')
    def destDir = new File(this.buildDir, 'generated/release/')
    doLast{
        println '开始解析对应的xml文件'
        destDir.mkdir()
        def releases = new XmlParser().parse(srcFile)
        releases.release.each{ releaseNode ->
            //解析每个reslease结点的内容
            def name = releaseNode.versionName.text()
            def versionCode = releaseNode.versiointCode.text()
            def versionInfo = releaseNode.versionInfo.text()
            //创建文件并写入结点数据
            def destFile = new File(destDir, "release-${name}.text")
            destFile.withWriter{ writer ->
                writer.write("${name} -> ${versionCode} -> ${versionInfo}")
            }
        }
    }
}

task handleReleaseFileTest(dependsOn: handleReleaseFile){
    //得到该目录
    def dir = fileTree(this.buildDir.path + 'generated/release/')
    doLast {
        dir.each {
            println 'the file name is:' + it
        }
        println '输出完成....'
    }
}
```

然后执行 `./gradlew handleReleaseFileTest`命令，可以得到如下输出：

```groovy
> Task :app:handleReleaseFile
开始解析对应的xml文件

> Task :app:handleReleaseFileTest
输出完成....
```

在我们执行测试任务的时候，也会执行相对应的依赖任务。之后在自定义的输出目录下面就会有相应的文件生成：

![image-20220110234942034](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220110234945.png)

上面提到的`release.xml`文件如下：

```xml
<release>
  <release>
    <versionCode>100</versionCode>
    <versionName>1.0.0</versionName>
    <versionInfo>App的第一个版本， 上线了一些基础功能</versionInfo>
  </release>
</release>
```

#### 2.通过Task输入输出指定

我们也可以通过Task来指定输入输出，使用这种方式来完成一个自动维护版本发布文档的gradle脚本。

```groovy
ext{
    versionName = '1.0.0'
    versionCode = '100'
    versionInfo = 'App的第一个版本， 上线了一些基础功能'
    destFile = file('release.xml')
    if (destFile != null && !destFile.exists()){
        destFile.createNewFile()
    }
}

task writeTask{
    //为task指定输入
    inputs.property('versionCode', this.versionCode)
    inputs.property('versionName', this.versionName)
    inputs.property('versionInfo', this.versionInfo)
    //为Task指定输出
    outputs.file this.destFile

    doLast {
        //将输入的内容写进去
        def data = inputs.getProperties()
        File file = outputs.getFiles().getSingleFile()
        //将map转化为实体对象
      def versionMsg = new VersionMsg(data)
      
      //将实体对象写入到xml中
   		....... 
   }
  
  task readTask{
    inputs.file destFile
    doLast {
      	//读取输入文件的内容并显示
        def file = inputs.files.singleFile
        println file.text
    }
}
  
  //最后对它进行测试
  task taskTest{
    dependsOn readTask, writeTask
    doLast {
        println '输入输出任务结束'
    }
}
```

在这里我们定义了readTask和writeTask，一个负责读取另一个则负责写入。在writeTask任务中指定了输出文件destFile，并把版本信息写入进去。最后在TaskTest任务重使用的dependdOn将两个task关联起来，这样就完成了这个例子--[releaseinfo.gradle](https://github.com/LebronXia/GradleTestDemo/blob/main/releaseinfo.gradle)

可是每一次执行都要我们手动执行，有没有可以直接在构建中就帮我们执行了这个任务？

```groovy
//挂载到build构建中去
project.afterEvaluate {project ->
    def buildTask = project.tasks.getByName('build')
    if (buildTask == null){
        throw GradleException('the build task is not found')
    }

    //buildTask.finalizedBy "writeTask"
    buildTask.doLast{
        writeTask.finalizedBy()
    }
}
```

#### 3.通过API指定执行顺序

我们还可以在闭包中通过`mustRunAfter`方法指定task的依赖顺序，它必须结合`dependesOn`强依赖进行配套使用。

```groovy
task taskX {
  //如果同时执行taskX、testY这2个task，会确保testY执行完之后才执行taskX这个task，用这个来保证执行顺序
    mustRunAfter "taskY"

    doFirst {
        println "this is taskX"
    }
}

task taskY {
    // 使用 mustRunAfter 指定依赖的（一至多个）前置 task
    // 也可以使用 shouldRunAfter 的方式，但是是非强制的依赖
		// shouldRunAfter taskA
    doFirst {
        println "this is taskY"
    }
}

task taskZ(dependsOn: [taskX, taskY]) {
    mustRunAfter "taskY"
    doFirst {
        println "this is taskZ"
    }
}
```

### Task类型

Task已经帮我们定义了很多Task Type供我们使用，平时用的时候多查看下官方文档[Gradle DSL API文档](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#N18930)中的Task types那一栏下。

Gradle本身还提供了一些已有的Task供我们使用，比如Copy、Delete等。因此我们定义Task的时候是可以继承已有的Task，如下实例代码：

```groovy
// 将 doc 复制到 build/target 目录下
task copyDocs(type: Copy) {
    from 'src/main/doc'
    into 'build/target/doc'
}

// 删除根目录下的 build 文件
task clean(type: Delete) {
    delete rootProject.buildDir
}
```

另外我们也可以自定义task来引用:

```groovy
class SayHelloTask extends DefaultTask {
    
    String msg = "default name";
    int age = 20

  //@TaskAction 表示该Task要执行的动作,即在调用该Task时，sayhello()方法将被执行
    @TaskAction
    void sayHello() {
        println "Hello $msg ! Age is ${age}"
    }
}

// hello使用了默认的message值
task hello(type:SayHelloTask)

// 重新设置了message的值
task hello1(type:SayHelloTask){
    message ="I am an android developer"
}
```

### 挂接自定Task到构建过程

如果想把自定义Task挂接到生命周期中间部分，而不是开头或者结束位置，又是如何实现？

学习这个最好是查看Tinker源码，在里面定义了如何挂接自定义Task到构建过程中:

![image-20220111160233216](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220111231212.png)

可以转化为构建图理解起来更清晰，这样自定义的task就放在了`processReleaseManifest`之后`processReleaseResources`之前了。

![image-20220111160648793](https://raw.githubusercontent.com/LebronXia/picture-warehouse/main/20220111160651.png)





**参考**

[深入理解Android之Gradle](https://blog.csdn.net/Innost/article/details/48228651)

[深度探索 Gradle 自动化构建技术（三、Gradle 核心解密）](https://juejin.cn/post/6844904132092903437)

[看完这一系列，彻底搞懂 Gradle](https://juejin.cn/post/6844903870091493384)

[搞定Groovy闭包这一篇就够了](https://www.jianshu.com/p/6dc2074480b8)

[Android Gradle学习(三)：Task进阶学习](https://www.jianshu.com/p/c45861426eba)

[Project文档](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html)

[Task文档](https://docs.gradle.org/current/javadoc/org/gradle/api/Task.html)

[TaskContainer API](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2Ftasks%2FTaskContainer.html)

