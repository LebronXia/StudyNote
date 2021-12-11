# Context

- Context 的使用上如何避免内存泄漏
- Android应用中哪些是 Context，一个应用有多少个 Context？
- 如何跨进程拿 Context？如 Activity 还没启动的时候如何拿 Context？
- Application中可以显示Dialog么？为什么？

**Context数量 = Activity数量 + Service数量 + 1 **

- 不要让生命周期长的对象引用Activity Context，即保证引用activity的对象要与activity本身生命周期是一样的

- 对于生命周期长的对象，可以使用Application Context

- 避免非静态的内部类，尽量使用静态类，避免生命周期问题，注意内部类对外部对象引用导致的生命周期变化

AMS 在启动 Activity 的时候，会构建表示 Activity 信息的 ActivityRecord 对象，其构造函数中会实例化 Token 对象

AMS 在接着上一步之后，会利用创建的 Token 构建 AppWindowContainerController 对象，最终将 Token 存储到 WMS 中的 mTokenMap 中

WMS 在 addWindow 时，会根据当前 Window 对象的 Token 进行校验

[Android Context 熟悉还是陌生](https://www.jianshu.com/p/cc0bb2a71ee8)

[为什么不能使用 Application Context 显示 Dialog？](https://juejin.cn/post/6867390363020361742)

# Window

- Activity、View、Window 之间的关系
- View.post 为什么可以获取到 View 的宽高
- Activity中的Window的初始化和显示过程

Activity 是四大组件之一，也是我们的界面载体，可以展示页面；而 View 实际上就是一个一个的视图，这些视图可以搭载在一个 Layout 文件上，通过 Activity 的 `setContentView()` 方法传递给 Activity；Window 是一个窗体，每个 Activity 对应一个 Window，通常我们在代码中用 getWindow() 来获取它。

**Activity中的Window的初始化和显示过程**

1 Activity在其attach方法中创建了Window，实际上创建的是PhoneWindow，并通过各种回调等建立起与Activity的联系，我们在Activity中使用getWindow（）所得到的即是这个创建的Window。
2 在PhoneWindow中装载了一个顶级的View，即DecorView，它实际是一个FrameLayout，并通过各种主题样式选择加载不同的视图来填充DecorView，这部分被称作mContentRoot。
3 setContentView设置我们自己需要展示的视图，这部分视图被填充到了DecorView的一块子ViewGroup中，这块子ViewGroup被称作contentParent。
4 DecorView最终是通过WindowManager的addView方法添加到Window的，并且最终在Activity的onResume方法执行后显示出来。
5 在使用requestWindowFeature来设置样式时，实际上是调用了PhoneWindow的requestFeature方法，会将样式存储在Window的mLocalFeatures变量中，当installDecor时，会应用这些样式。也就是说，当需要通过requestWindowFeature来请求样式时，应该在setContentView方法之前调用，因为setContentView方法的调用会导致DecorView的创建并应用样式，如果在之后调用则会导致不会生效，因为此时DecorView已经创建完成了。



将我们调用 View.post 方法传入的 Runnable 发送到主线程的消息队列，消息是同步类型。Handler 的消息队列循环过程中，在一个消息执行完之后才会取下一个消息。因为这一特性，所以在异步消息没执行完之前，消息队列中的消息是不会执行的。所以调用了 HandlerActionQueue 的 executeActions 方法，发送到主线程消息队列的消息们不会被立即执行，等 performTraversals 方法执行完，也就是异步消息结束之后， HandlerActionQueue 的 executeActions 方法，发送到主线程消息队列的消息们才会被执行。ViewRoomImpl 的 performTraversals 方法注释 1 处，开始了 View 绘制流程，依次是测量 performMeasure、布局 performLayout 和绘制 performDraw，这三个方法走完，标志着我们的 UI 已经完成显示了。

[简析Window、Activity、DecorView以及ViewRoot之间的错综关系](https://www.jianshu.com/p/8766babc40e0)

[View.post 为什么可以获取到 View 的宽高](https://juejin.cn/post/6994044328918122510#heading-3)
[震惊！Android子线程也能修改UI？](https://www.jianshu.com/p/1b2ccd3e1f1f)

[Android Activity之Window的创建过程](https://blog.csdn.net/qq_28261343/article/details/78817184)

# Activity Service 四大组件

- 对Activity的理解是什么
- 启动模式以及使用场景
- Service启动方式以及如何停止
- [startService 和 bingService区别](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fd870f99b675c) 
- 广播的使用场景，原理

**每个 Activity 包含了一个 Window 对象，这个对象是由 PhoneWindow 做的实现。而 PhoneWindow 将 DecorView 作为了一个应用窗口的根 View，这个 DecorView 又把屏幕划分为了两个区域：一个是 TitleView，一个是 ContentView，而我们平时在 Xml 文件中写的布局正好是展示在 ContentView 中的。**

[Activity的四种启动模式应用场景](https://blog.csdn.net/black_bird_cn/article/details/79764794)

[Android中startService和bindService的区别](https://www.jianshu.com/p/d870f99b675c)

[23 个安卓重难点突破，带你吃透 Service 知识点](https://juejin.cn/post/6844903986131271688#heading-17)

# OOM和内存泄漏

- 为什么会出现OOM
- 哪些原因会导致OOM
- debug包有什么修改方式使不出现oom
- 有哪些原因会引起内存泄漏
- 内存泄漏有设么方式检测？用过哪些工具，其中的原理是什么

用ActivityLifecycleCallbacks接口来检测Activity生命周期 ，主要是在Activity的&**onDestroy**方法中，手动调用 GC，然后利用ReferenceQueue+WeakReference，来判断是否有释放不掉的引用，然后结合dump memory的hpof文件, 用[HaHa](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fsquare%2Fhaha)分析出泄漏地方

[Android性能优化：关于 内存泄露 的知识都在这里了！](https://www.jianshu.com/p/97fb764f2669)

[Java内存问题 及 LeakCanary 原理分析](https://juejin.cn/post/6844903583129796622)

[Android Handler：详解 Handler 内存泄露的原因](https://www.jianshu.com/p/ed9e15eff47a)

# Handle机制

- Looper.loop()为什么不会阻塞主线程
- IdHandler(闲时机制）；postDelay()的具体实现
- post()与sendMessage()区别
- 用Handler需要注意什么问题，怎么解决的?

1. Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗？
2. 主线程的消息循环机制是什么（死循环如何处理其它事务）？
3. ActivityThread 的动力是什么？（ActivityThread执行Looper的线程是什么）
4. Handler 是如何能够线程切换，发送Message的？（线程间通讯）
5. 子线程有哪些更新UI的方法。
6. 子线程中Toast，showDialog，的方法。（和子线程不能更新UI有关吗）
7. 如何处理Handler 使用不当导致的内存泄露？

[**Android消息机制**](https://github.com/interviewandroid/AndroidInterView/blob/master/study/framework/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6.md)

说到Handler，就不得不提与之密切相关的这几个类：Message、MessageQueue，Looper。

- **Message**。 Message中有两个成员变量值得关注：target和callback。target其实就是发送消息的Handler对象，callback是当调用handler.post(runnable)时传入的Runnable类型的任务。post事件的本质也是创建了一个Message，将我们传入的这个runnable赋值给创建的Message的callback这个成员变量。
- **MessageQueue**。消息队列很明显是存放消息的队列，值得关注的是MessageQueue中的next()方法，它会返回下一个待处理的消息。
- **Looper**。Looper消息轮询器其实是连接Handler和消息队列的核心。首先我们都知道，如果想要在一个线程中创建一个Handler，首先要通过Looper.prepare()创建Looper，之后还得调用Looper.loop()开启轮询。我们着重看一下这两个方法。

prepare()。这个方法做了两件事：首先通过ThreadLocal.get()获取当前线程中的Looper,如果不为空，则会抛出一个RunTimeException，意思是一个线程不能创建2个Looper。如果为null则执行下一步。第二步是创建了一个Looper，并通过ThreadLocal.set(looper)。将我们创建的Looper与当前线程绑定。这里需要提一下的是消息队列的创建其实就发生在Looper的构造方法中。

loop()。这个方法开启了整个事件机制的轮询。它的本质是开启了一个死循环，不断的通过MessageQueue的next()方法获取消息。拿到消息后会调用msg.target.dispatchMessage()来做处理。其实我们在说到Message的时候提到过，msg.target其实就是发送这个消息的handler。这句代码的本质就是调用handler的dispatchMessage()。Handler。上面做了这么多铺垫，终于到了最重要的部分。Handler的分析着重在两个部分：发送消息和处理消息

- Handler。上面做了这么多铺垫，终于到了最重要的部分。Handler的分析着重在两个部分：发送消息和处理消息

发送消息。其实发送消息除了sendMessage之外还有sendMessageDelayed和post以及postDelayed等等不同的方式。但它们的本质都是调用了sendMessageAtTime。在sendMessageAtTime这个方法中调用了enqueueMessage。在enqueueMessage这个方法中做了两件事：通过msg.target = this实现了消息与当前handler的绑定。然后通过queue.enqueueMessage实现了消息入队。

处理消息。消息处理的核心其实就是dispatchMessage()这个方法。这个方法里面的逻辑很简单，先判断msg.callback是否为null，如果不为空则执行这个runnable。如果为空则会执行我们的handleMessage方法。

IdleHandler是主线程在开始加载页面完成后调用的方法，可以提高性能

**Loop 死循环，为什么没有阻塞 主线程**

- 真正会卡死主线程的操作是在回调oncreat、onstart、onresume中，会导致卡帧，甚至ANR
- 涉及linux pipe机制,主线程中messaequeue没有消息，会阻塞loope 中queue.next，会释放CPU资源。直到下个消息往管道里写数据时会唤起主线程工作。
- epoll机制多路复用机制，监控多个描述符，某个描述符就位时，立刻通知读写

[Handler 面试相关](https://blog.csdn.net/pengyu1801/article/details/104718819/)

# 性能优化

- UI方面的优化



**WebView启动优化**

1. 我们可以定义全局WebView对象并提前初始化，如果需要使用WebView加载页面直接使用初始化好的WebView对象就可以了。WebView第一次创建比较耗时，可以预先创建WebView，提前将其内核初始化。

2. 使用WebView缓存池，用到WebView的地方都从缓存池取，缓存池中没有缓存再创建，注意内存泄漏问题。

3. 本地预置html和css，WebView创建的时候先预加载本地html，之后通过js脚本填充内容部分。
4. webview 可以加载网络资源，那么也是可以加载本地的资源，在apk 启动的时候，我们可以把整个前端代码文件下载解压到本地的文件路径中，然后通过file：///...index.html 去打开本地的资源
5. 可以把前端的ajax请求提前到和页面加载同时进行，由客户端请求数据，等到H5加载完毕，直接向客户端索要即可
6. 图片资源的拉取是最为耗时的，一个比较好的解决方案就是先加载并展示非图片内容，延迟这些图片的加载，以提升用户体验。WebView有一个setBlockNetworkImage(boolean)方法，该方法的作用是是否屏蔽图片的加载

[Android WebView最佳优化（WebView池）](https://blog.csdn.net/u011082160/article/details/118245494)

[android性能优化(三)之Webview优化](https://blog.csdn.net/qingtiantianqing/article/details/100066884)

[android性能优化(二)之卡顿优化](https://blog.csdn.net/qingtiantianqing/article/details/100066873)

[面试官：今日头条启动很快，你觉得可能是做了哪些优化](https://juejin.cn/post/6844903958113157128#heading-17)

[今日头条App 页面秒开方案详解](https://mp.weixin.qq.com/s/KwvWURD5WKgLKCetwsH0EQ)

# Android多进程

[**Java 中的进程通信方式**](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqq_38225558%2Farticle%2Fdetails%2F87118551)：管道(pipe)、有名管道(namedpipe)、信号量(semophore) 、套接字(socket)、消息队列(messagequeue) 、共享内存(shared memory) 、信号(sinal)

[**Android 中的进程通信方式**](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu011240877%2Farticle%2Fdetails%2F72863432)：AIDL(基于 Binder)、Messenger (基于 Binder)、Socket、Binder (基于 mmp 共享内存)、文件

# UI适配方案

- 对于 UI 稿的 px 是如何适配的

[一文读懂 Android 主流屏幕适配方案](https://juejin.cn/post/6999445137491230728)

[今日头条屏幕适配方案终极版正式发布!](https://juejin.cn/post/6844903697000972295)

# Bitmap

- 下载一张很大的图，如何保证不 oom？
- biamap复用
- Bitmap内存大小，注意事项，如何优化

**内存大小 = （设备屏幕dpi / 资源所在目录dpi）^ 2 × 图片原始宽 × 图片原始高 × 像素大小**

图片内存大小以是Bitmap的实际尺寸大小为准，而不是图片显示大小

- 对图片质量进行压缩
- 对图片尺寸进行压缩

ALPHA_8：表示8位Alpha位图，每像素占1byte内存；

RGB_565：表示R为5位，G为6位，B为5位，一共16位，每像素占2byte内存；

ARGB_4444：表示16位位图，每像素占2byte内存（poor quality - Android Deprecated）；

ARGB_8888：表示32位ARGB位图，每像素占4byte内存（Recommended）



利用的就是inBitmap指定将要重复利用的Bitmap对象的内存。同时需要指定inMutable=true表示对象是可变的。如果inPreferredConfig = android.graphics.Bitmap.Config.HARDWARE，inMutable属性永远为false。

**Android图片压缩**

 1. 质量压缩

 2. 尺寸压缩

 3. 采样率压缩

 4. 通过JIN调用libjpeg库压缩

 5. 图片格式

```java
 public static Bitmap compressInSampleSize(Resources resources, int resId, int >reqWidth, int reqHeight) {
       BitmapFactory.Options options = new BitmapFactory.Options();
       options.inJustDecodeBounds = true;
       Bitmap bitmap = BitmapFactory.decodeResource(resources, resId, options);
       options.inSampleSize = getInSampleSize(bitmap, reqWidth, reqHeight);
       options.inJustDecodeBounds = false;
       bitmap.recycle();
       Bitmap resultBitmap = BitmapFactory.decodeResource(resources, resId, options);
       return resultBitmap;
   }

```

[Android 图片压缩最常用的几种方法!](https://www.jianshu.com/p/1f69238b8874)


[Android性能优化系列之Bitmap图片优化](https://blog.csdn.net/u012124438/article/details/66087785)

[Bitmap的inBitmap使用](https://my.oschina.net/u/3863980/blog/3019921)

# WebView 与 JS 交互方式

- shouldOverrideUrlLoading、onJsPrompt使用有啥区别 

[最全面总结 Android WebView与 JS 的交互方式](https://www.jianshu.com/p/345f4d8a5cfa)

# APP打包流程

- aapt 工具打包资源文件，生成 R.java 文件

- aidl 工具处理 AIDL 文件，生成对应的 .java 文件

- javac 工具编译 Java 文件，生成对应的 .class 文件

- 把 .class 文件转化成 Davik VM 支持的 .dex 文件

- apkbuilder 工具打包生成未签名的 .apk 文件

- jarsigner 对未签名 .apk 文件进行签名

- zipalign 工具对签名后的 .apk 文件进行对齐处理

## Apk签名

- APK 为什么要签名
- 是否了解过具体的签名机制

Android 为了确认 apk 开发者身份和防止内容的篡改，设计了一套 apk 签名的方案保证 apk 的安全性，即在打包时由开发者进行 apk 的签名，在安装 apk 时Android 系统会有相应的开发者身份和内容正确性的验证，只有验证通过才可以安装 apk，签名过程和验证的设计就是基于非对称加密的思想。
 Android 在 7.0 以前使用的一套签名方案：在 apk 根目录下的 META-INF/ 文件夹下生成签名文件，然后在安装时在系统的 PackageManagerService 里进行签名文件的验证。
 从 7.0 开始，Android 提供了新的 V2 签名方案：利用 apk(zip) 压缩文件的格式，在几个原始内容区之外增加了一块用于存放签名信息的数据区，然后同样在安装时在系统的 PackageManagerService 里进行 V2 版本的签名验证，V2 方案会更安全、使校验更快安装更快。
 当然 V2 签名方案会向后兼容，如果没有使用 V2 签名就会默认走 V1 签名方案的验证过程。

V1是签名以文件的形式存在于这个版本的apk包就是一个标准的zip包

V2是对整个zip包进行签名，而且在zip包中增加了 一个apk signatureblock，里面保存签名信息

# Binder机制

- 介绍下 Binder 机制，与内存共享机制有什么区别
- Binder如何跨进程

Binder是基于C/S架构的，简单解释下C/S架构，是指客户端(Client)和服务端(Server)组成的架构，Client端有什么需求，直接发送给Server端去完成，架构清晰明朗，Server端与Client端相对独立，稳定性较好；而共享内存实现方式复杂，没有客户与服务端之别， 需要充分考虑到访问临界资源的并发同步问题，否则可能会出现死锁等问题；从这稳定性角度看，Binder架构优越于共享内存。

1. 数据先从发送方的缓存区拷贝到内核开辟的缓存区中，再 从内核缓存区拷贝到接收方的缓存区，一共一次拷贝

2. Binder 基于 C/S 架构 ，Server 端与 Client 端相对独立，稳定性较好

3. Binder 机制为 每个进程分配了 UID/PID 且在 Binder 通信时会根据 UID/PID 进行有效性检测

**Binder机制主要的流程是这样的**

服务端通过Binder驱动在ServiceManager中注册我们的服务。

客户端通过Binder驱动查询在ServiceManager中注册的服务。

ServiceManager通过Binder驱动返回服务端的代理对象。

客户端拿到服务端的代理对象后即可进行进程间通信。

[为什么 Android 要采用 Binder 作为 IPC 机制](https://www.zhihu.com/question/39440766/answer/89210950)

[Carson带你学Android：图文详解Binder跨进程通信原理](https://www.jianshu.com/p/4ee3fd07da14)

# 事件分发

- View、ViewGroup的事件传递机制，如何解决滑动冲突？ 回答如何滑动-冲突最好是举出实际的场景和怎么解决的

整个View的事件转发流程是：

View.dispatchEvent->View.setOnTouchListener->View.onTouchEvent

在dispatchTouchEvent中会进行OnTouchListener的判断，如果OnTouchListener不为null且返回true，则表示事件被消费，onTouchEvent不会被执行；否则执行onTouchEvent。

```java
  public boolean dispatchTouchEvent(MotionEvent event) {
        if (!onFilterTouchEventForSecurity(event)) {
            return false;
        }
 
        if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
                mOnTouchListener.onTouch(this, event)) {
            return true;
        }
        return onTouchEvent(event);
```

> setOnLongClickListener和setOnClickListener不是只能执行一个

**ViewGroup事件传递**

Android事件分发是先传递到ViewGroup，再由ViewGroup传递到View的。

2. 在ViewGroup中可以通过onInterceptTouchEvent方法对事件传递进行拦截，onInterceptTouchEvent方法返回true代表不允许事件继续向子View传递，返回false代表不对事件进行拦截，默认返回false。

3. 子View中如果将传递的事件消费掉，ViewGroup中将无法接收到任何事件

**解决滑动冲突**

1、如果ViewGroup找到了能够处理该事件的View，则直接交给子View处理，自己的onTouchEvent不会被触发；

2、可以通过复写onInterceptTouchEvent(ev)方法，拦截子View的事件（即return true），把事件交给自己处理，则会执行自己对应的onTouchEvent方法

3、子View可以通过调用getParent().requestDisallowInterceptTouchEvent(true);  阻止ViewGroup对其MOVE或者UP事件进行拦截；

[Android事件分发机制详解：史上最全面、最易懂](https://www.jianshu.com/p/38015afcdb58)

# 自定义View

- View、ViewGroup的绘制流程
- Android子线程也能修改UI
- requestLayout，invalidate 的区别

**requestLayout，invalidate 的区别**

1. requeLayout() : 控件会重新执行 onMesure() onLayout() ,比如 ScrollView中有LinearLaout ，LinearLayout里面有纵向排列的ImageView和TextView,那么假如ImageView的长宽发生了变化，而要立即在手机上显示这个变化的话，就可调用 imageView.requestLayout();这样的话ScrollView 会重新执行onMesure()这个方法会确定控件的大小然后在确定出自己的宽高，最后在执行onLayout()，这个方法是对所有的子控件进行定位的。他只调用measure()和layout()过程，不会调用draw()。

2. invalidate() :是自定义View 的时候，重新执行onDraw()方法，当view只在内容和可见度方面发生变化时调用。

3. layout()：对控件进行重新定位执行onLayout()这个方法，比如要做一个可回弹的ScrollView，思路就是随着手势的滑动子控件滑动，那么我们可以将ScrollView的子控件调用layout（l,t,r,b）这个方法就行了。

刷新原理：View 的 requestLayout 和 ViewRootImpl##setView 最终都会调用 ViewRootImpl 的 requestLayout 方法。然后通过 scheduleTraversals 方法提交绘制任务，然后再通过DisplayEventReceiver向底层请求vsync垂直同步信号，当vsync信号来的时候，通过JNI回调回来，再通过Handler往消息队列post一个异步任务，最终是ViewRootImpl去执行绘制任务，最后调用performTraversals方法，完成绘制。

最终**performTraversals()**方法触发了View的绘制。该方法内部，依次调用了performMeasure(),performLayout(),performDraw(),将View的measure，layout，draw过程，从顶层View分发了下去。

可以看到一般情况下View的draw流程分为四步：

1. 绘制背景
2. 绘制自身内容（onDraw）
3. 遍历子View，调用其draw方法把绘制过程分发下去（dispatchDraw）
4. 绘制装饰（onDrawForeground）

**自定义View注意事项**

1. 让View支持wrap_conent
2. 让View支持padding
3. 尽量避免使用Handler，一般都可以用View自带的post方法代替
4. 在onDeatchFromWindow时，停止View的动画或线程（如果有的话）
5. 如果存在嵌套滑动，处理好滑动冲突

**UI原理**

1. Activity的attach 方法里创建PhoneWindow。

2. onCreate方法里的 setContentView 会调用PhoneWindow的setContentView方法，创建DecorView并且把xml布局解析然后添加到DecorView中。

3. 在onResume方法执行后，会创建ViewRootImpl，它是最顶级的View，是DecorView的parent，创建之后会调用setView方法。

4. ViewRootImpl 的 setView方法，会将PhoneWindow添加到WMS中，通过 Session作为媒介。  setView方法里面会调用requestLayout，发起绘制请求。

5. requestLayout 一旦发起，最终会调用 performTraversals 方法，里面将会调用View的三个measure、layout、draw方法，其中View的draw 方法需要一个传一个Canvas参数。

6. 最后分析了软件绘制的原理，通过relayoutWindow 方法将Surface跟当前Window绑定，通过Surface的lockCanvas方法获取Surface的的Canvas，然后View的绘制就通过这个Canvas，最后通过Surface的unlockCanvasAndPost 方法提交绘制的数据，最终将绘制的数据交给SurfaceFlinger去提交给屏幕显示。。

[[Android View绘制13问13答](https://www.cnblogs.com/punkisnotdead/p/5181821.html)](https://www.cnblogs.com/punkisnotdead/p/5181821.html)

[面试官问你：自定义View跟绘制流程懂吗？帮你搞定面试官](https://blog.csdn.net/c10wtiybq1ye3/article/details/103415297)

# Recyleview

- 按下列表item的button 同时上下滑动RecyclerView 事件是怎么处理的
- RecyclerView加载多图速度优化方案
- RecyclerView的缓存，局部刷新用过么

**RecycleView四级缓存**

- **Scrap**:屏幕内部的 ItemView,通过数据集的 Position 找到的，**可以直接复用不需要绑定**
- **Cache**:移出屏幕的 ItemView 放入一个CacheView（默认个数为2），就是比屏幕多两个 ItemView， 方便来回翻少量的View，**可以直接复用不需要绑定**
- **ViewCacheExtension**:用户自定义的缓存机制

> **ViewCacheExtension Example **
>
> 
>
> 广告夹杂在 RecycleView 内，并且不会发生变化，广告和内容分开请求
>
> - 广告卡片
> - - 每页一共有四个广告
>   - 短期内不会发生变化
> - Cache 只关心 position，不关心 view type
> - RecycleViewPool 只关心 view type，每次都要重新绑定
> - 解决方法：在 ViewCacheExtension 保存四个广告的 Card

- **RecycleViewPool**:被废弃的 ItemView ，内部的 data 都是 dirty 的。通过ViewType来重新Bind数据的，RecycledViewPool默认的缓存数量是5个



`notifyItemRangeChanged`如果传入null表示更新全部，这时就要重写`onBindViewHolder`

```java
//在第一个OnBindViewHolder里面我们需要判断下payloads是否为空，如果为空的话执行默认的OnBindViewHolder方法，
//如果不为空的话，执行我们自己的局部刷新方法

@Override
    public void onBindViewHolder(TestDownLoadHolder holder, int position, List<Object> payloads) {
        XLog.e("--------------------------has  payloads");
        if (payloads.isEmpty()) {
            XLog.e("--------------------------no  payloads");
            onBindViewHolder(holder, position);
        } else {
            XLog.e("--------------------------false  payloads");
            final AppInfoBean appInfoBean = data.get(position);
            if (appInfoBean != null) {
                holder.mPbDownProgress.setProgress(appInfoBean.getProgress());
                holder.mBtDownLoad.setText(appInfoBean.getDownloadPerSize());
            }
        }
    }


//考虑要不要用
RecyclerView.ViewHolder viewHolder = mRecyclerView.findViewHolderForAdapterPosition(i);
            if (viewHolder != null && viewHolder instanceof ItemHolder) {
                ItemHolder itemHolder = (ItemHolder) itemHolder 
                    itemHolder.mButton.togglestate();
                }
            }
```



[让你彻底长我RecyclerView的缓存机制](http://www.360doc.com/content/19/0712/11/36367108_848240455.shtml)

[阿里3轮面试都问了RecyclerView](https://www.jianshu.com/p/eabb00c500ef)

# Gradle

- Gradle 的工作原理
- Gradle的实现，gradle中task的生命周期。
- gradle生命周期，task，插件

[看完这一系列，彻底搞懂 Gradle](https://juejin.cn/post/6844903870091493384)

[Android 修炼手册】Gradle 篇 -- Gradle 的基本使用](https://blog.csdn.net/weixin_42118423/article/details/112138400)

初始化阶段，Gradle为项目创建了一个Project实例。给定的构建脚本只定义了一个项目，在多项目构建中，这个构建阶段变得更加重要。根据正在执行的项目，Gradle找出哪些项目依赖需要参与到构建中。

> 注意：在这个构建阶段当前已有的构建脚本代码都不会被执行。

配置阶段，Gradle构造了一个模型来表示任务，并参与到构建中来。增量式构建特性决定了模型中的task是否需要被运行。这个阶段非常适合与为项目或执行task设置所需的配置。

> 注意：项目每一次构建的任何配置代码都可以被执行——即使你只执行gradle tasks

执行阶段，所有的task都应该以正确的顺序被执行。执行顺序是由它们的依赖决定的。如果任务被认为没有修改过，将被跳过，这个牵涉到增量式构建，

gradle 构建分为三个阶段初始化阶段
初始化阶段主要做的事情是有哪些项目需要被构建，然后为对应的项目创建 Project 对象

配置阶段
配置阶段主要做的事情是对上一步创建的项目进行配置，这时候会执行 build.gradle 脚本，并且会生成要执行的 task

执行阶段
执行阶段主要做的事情就是执行 task，进行主要的构建工作

# 动画

- 属性动画原理

Animation 通过PropertyValuesHolder 来更新对象的目标属性，如果用户没有设定目标属性的Property对象，那么会通过反射的形式调用目标属性的setter方法来更新属性值；否则，通过Property的set方法来设置属性值，这个属性值则通过KeyFrameSet的计算得到，而KeyFrameSet又是通过时间插值器和类型估值器来计算的，在动画执行的过程中不断计算当前时刻目标属性的值，然后 更新属性值来达到动画效果。

# AOP

- ASM
- AspectJ

Android AOP就是通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，提高开发效率。

- ASM是一个框架/库，它为您提供了一个API来操纵现有的字节码和/或轻松生成新的字节码
- AspectJ是Java语言之上的一种语言扩展，具有它自己的语法，专门用于扩展具有面向方面编程概念的Java运行时的功能。它包含一个编译器/编织器，它可以在编译时或运行时运行

# 序列化

- Serializable 与 Parcelable 的区别？
- Serializable 中的 serialVersionUID 作用，如果修改了一个值，这个ID是否会改变?

Serializable 接口是一种标识接口（marker interface），这意味着无需实现方法，Java便会对这个对象进行高效的序列化操作。

这种方法的缺点是使用了反射，序列化的过程较慢。这种机制会在序列化的时候创建许多的临时对象，容易触发垃圾回收。

Parcelable方式的实现原理是将一个完整的对象进行分解，而分解后的每一部分都是Intent所支持的数据类型，这样也就实现传递对象的功能了

**选择序列化方法的原则**

1. 在使用内存的时候，Parcelable比Serializable性能高，所以推荐使用Parcelable。
2. Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。
3. Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点，但此时还是建议使用Serializable 。

Java的序列化机制是通过在运行时判断类的serialVersionUID来验证版本一致性的。在进行反序列化时，JVM会把传来的字节流中的serialVersionUID与本地相应实体（类）的serialVersionUID进行比较，如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常。

# MVVM

[是让人耳目一新的 Jetpack MVVM 精讲啊](https://zhuanlan.zhihu.com/p/139726434)

# 混淆

-  android混淆的步骤和原理

1. 压缩 ：检测并移除代码中无用的类、字段、方法和属性（Attribute）；

2. 优化：对字节码进行优化，移除无用的指令；

3. 混淆：使用a，b，c，d这样简短而无意义的名称，对类、字段和方法进行重命名；

4. 预检测：在Java平台上对处理后的代码进行预检，确保加载的class文件是可执行的；
   

**proguard 工作原理**

- 移除没有用到的代码，然后对代码里面的类、变量、方法重命名为人可读性很差的简短名字。

- Entry Point（入口点） :
  标识不会被处理的类和方法; 在压缩的步骤中，Proguard会从上述的Entry Point开始递归遍历，搜索哪些类和类的成员在使用，对于没有被使用的类和类的成员，就会在压缩段丢弃，在接下来的优化过程中，那些非Entry Point的类、方法都会被设置为private、static或final，不使用的参数会被移除，此外，有些方法会被标记为内联的，在混淆的步骤中，Proguard会对非Entry Point的类和方法进行重命名。
  

# ANR

- ANR发生的原理是什么， 怎么排查

APP发生卡顿，卡顿超过了阈值，就会报ANR

ANR触发流程，可以比喻为埋炸弹和拆炸弹的过程，
 以启动Service为例，Service的onCreate方法调用之前会使用Handler发送延时10s的消息，Service 的onCreate方法执行完，会把这个延时消息移除掉。
 假如Service的onCreate方法耗时超过10s，延时消息就会被正常处理，也就是触发ANR，会收集cpu、堆栈等信息，弹ANR Dialog。

## ANR监控

**ANRWatchDog**

1. 开启一个线程，死循环，循环中睡眠5s

2. 往UI线程post 一个Runnable，将_tick 赋值为0，将 _reported 赋值为false

3. 线程睡眠5s之后检查_tick和_reported字段是否被修改

4. 如果_tick和_reported没有被修改，说明给主线程post的Runnable一直没有被执行，也就说明主线程卡顿至少5s**（只能说至少，这里存在5s内的误差）**。

5. 将线程堆栈信息输出

**抓取系统traces.txt 上传**

**ANR错误出现原因**：只有当应用程序的UI线程响应超时才会引起ANR 超时产生的原因包括：①当前事件没有机会处理，例如UI线程正在响应另外的事件，当前事件被某个事件给阻塞掉了；②当前事件正在处理 但是由于耗时太长没有能及时的完成。其他原因：③在BroadcastReceiver里做耗时的操作或计算；④CPU使用过高；⑤发生了死锁；⑥耗时操作的动画需要大量的计算工作，可能导致CPU负载过重。

ANR错误定位——如果开发机器上出现ANR问题时，系统会生成一个traces.txt的文件放在/data/anr下，最新的ANR信息在最开始部分。通过adb命令将其导出到本地，输入以下字符：

$adb pull data/anr/traces.txt 

Traceview - 系统性能分析工具，用于定位应用代码中的耗时操作

分析主线程堆栈、cpu、锁信息等

## 死锁监控

1. 获取blocked状态的线程
2. 获取该线程想要竞争的锁（native层函数）
3. 获取这个锁被哪个线程持有（native层函数）
4. 有了关系链，就可以找出造成死锁的线程

[给你一个Demo 你如何快速定位ANR](https://github.com/interviewandroid/AndroidInterView/blob/master/android/anr.md)

[卡顿、ANR、死锁，线上如何监控](https://juejin.cn/post/6973564044351373326#heading-27)

# APP卡顿

[面试官又来了：你的app卡顿过吗？](https://juejin.cn/post/6844903949560971277)

**应用卡顿，原因一般都可以认为是Handler处理消息太耗时导致的**，细分的原因可能是方法本身太耗时、算法效率低、cpu被抢占、内存不足、IPC超时等等。

被问到如何监控App卡顿，统计方法耗时，我们可以从源码开始切入，

## 卡顿监控

**Looper.printer** 

**字节码插桩**

 真实操作中可以采用插桩的方式，给对应范围的方法起止位加入日志，并计算方法的执行耗时，后续统计出耗时排行，然后针对方法再进行优化。UI方面也可以采用一些工具来发现过度绘制，以及绘制耗时，然后再根据统计情况来优化UI绘制卡顿。 优化的方式也比较多，比如IO集中，加入缓存，降低联网频次，合并网络请求，自定义UI尽可能扁平。控制锁的使用，防止锁循环，过度锁争夺，死锁，必要的情况下可以采用CAS模式。

# 插件化



# 热修复

- PathClassLoader与DexClassLoader的区别
- Android热修复原理，tinker的patch文件如何生成，patch文件是全部加载dex文件首部么？

只能加载内存中已经安装的apk中的dex，而后者可以加载sd卡中的apk/ja

假设现在代码中的某一个类或者是某几个类有bug，那么我们可以在修复完bug之后，可以将这些个类打包成一个补丁文件，然后通过这个补丁文件封装出一个Element对象，并且将这个Element对象插到原有dexElements数组的最前端，这样当DexClassLoader去加载类时，优先会从我们插入的这个Element中找到相应的类，虽然那个有bug的类还存在于数组中后面的Element中，但由于双亲加载机制的特点，这个有bug的类已经没有机会被加载了，这样一个bug就在没有重新安装应用的情况下修复了。

**Tinker思路**

通过修复好的class.dex和原有的class.dex比较差生差量包补丁文件patch.dex,在手机上这个patch.dex又会和原有的class.dex合并生成新的文件fix_class.dex,用着新的fix_class.dex整体替换原有的dexPathList的中的内容，可以说是从根本上把bug给你干掉了。

PathClassLoader：只能加载已经安装到Android系统中的apk文件（/data/app目录），是Android默认使用的类加载器。

DexClassLoader：可以加载任意目录下的dex/jar/apk/zip文件，比PathClassLoader更灵活，是实现热修复的重点。

[Android 插件化和热修复知识梳理](https://blog.csdn.net/TOYOTA11/article/details/78660511)

# 崩溃收集

- 怎防止程序崩溃，如果已经到了Thread.UncaughtExceptionHandler是否可以让程序继续运行。

了解下Android 的异常处理机制，在我们未设置Thread.UncaughtExceptionHandler之前，系统会默认设置一个。ZygoteInit.zygoteInit()，调用了` Process.killProcess(Process.myPid()); System.exit(10);`,触发了进程结束逻辑，也就导致了程序停止运行。

通过在主线程里面发送一个消息，捕获主线程的异常，并在异常发生后继续调用`Looper.loop`方法，使得主线程继续处理消息。

对于子线程的异常，可以通过`Thread.setDefaultUncaughtExceptionHandler`来拦截，并且子线程的停止不会给用户带来感知。

对于在生命周期内发生的异常，可以通过替换`ActivityThread.mH.mCallback`的方法来捕获，并且通过`token`来结束Activity或者直接杀死进程。但是这种办法要适配不同SDK版本的源码才行，所以慎用，需要的可以看文末Cockroach库源码。

Native Crash捕获

程序在遇到不可恢复的错误时会触发崩溃处理机制让程序退出，如除零、段地址错误等。而linux把这些中断处理，统一为信号量。

对异常异常的捕捉处理，在native层处理。

函数运行在用户态，当遇到系统调用、中断或是异常的情况时，程序会进入内核态。信号涉及到了这两种状态之间的转换

![image-20211029152427895](../../../../Library/Application Support/typora-user-images/image-20211029152427895.png)

- 设置额外栈空间

  SIGSEGV 很有可能是栈溢出引起的

- 兼容其他 signal 处理

  某些信号可能在之前已经被安装过信号处理函数，而 sigaction 一个信号量只能注册一个处理函数，这意味着我们的处理函数会覆盖其他人的处理信号。

- 防止死锁或者死循环

  

一般的出现崩溃信号，Android系统默认缺省操作是直接退出我们的程序。但是系统允许我们给某一个进程的某一个特定信号注册一个相应的处理函数（signal），即对该信号的默认处理动作进行修改。因此NDK Crash的监控可以采用这种信号机制，捕获崩溃信号执行我们自己的信号处理函数从而捕获NDK Crash。

[Android平台Native奔溃捕获机制及实现](https://www.jianshu.com/p/fbf910bcb38d)

[异常处理 - Native 层的崩溃捕获机制及实现](https://www.jianshu.com/p/6c751007bb3b)

[能否让APP永不崩溃](https://juejin.cn/post/6904283635856179214)

[App怎么做才能永不崩溃](https://juejin.cn/post/6925702262102687758)

[监控Java层和JNI Native层Crash](https://www.jianshu.com/p/4278915847b6)

# 启动流程

- Activity启动流程
- Launcher启动流程
- 如果进程不存在请求zygote fork出进程。这里使用的不是Binder，是socket。为什么不用bind

[【凯子哥带你学Framework】Activity启动过程全解析](https://blog.csdn.net/zhaokaiqiang1992/article/details/49428287)

**Launcher启动流程**

1. 点击桌面应用图标，Launcher进程将启动Activity（MainActivity）的请求以Binder的方式发送给了AMS。
2. AMS接收到启动请求后，交付ActivityStarter处理Intent和Flag等信息，然后再交给ActivityStackSupervisior/ActivityStack 处理Activity进栈相关流程。同时以Socket方式请求Zygote进程fork新进程。
3. Zygote接收到新进程创建请求后fork出新进程。
4. 在新进程里创建ActivityThread对象，新创建的进程就是应用的主线程，在主线程里开启Looper消息循环，开始处理创建Activity。
5. ActivityThread利用ClassLoader去加载Activity、创建Activity实例，并回调Activity的onCreate()方法。这样便完成了Activity的启动。

**Activity启动流程**

1. Activity1调用startActivity，实际会调用Instrumentation类的execStartActivity方法，Instrumentation是系统用来监控Activity运行的一个类，Activity的整个生命周期都有它的影子。（1- 4）

2. 通过跨进程的binder调用，进入到ActivityManagerService中，其内部会处理Activity栈，通知Activity1 Pause，Activity1 执行Pause 后告知AMS。（5 - 29）

3. 在ActivityManagerService中的startProcessLocked中调用了Process.start()方法。并通过连接调用Zygote的native方法forkAndSpecialize，执行fork任务。之后再通过跨进程调用进入到Activity2所在的**进程**中。（30 - 36）

4.  ApplicationThread是一个binder对象，其运行在binder线程池中，内部包含一个H类，该类继承于类Handler。主线程发起bind Application，AMS 会做一些配置工作，然后让主线程 bind ApplicationThread，ApplicationThread将启动Activity2的信息通过H对象发送给**主线程**。发送的消息是EXECUTE_TRANSACTION，消息体是一个 ClientTransaction，即 LaunchActivityItem。主线程拿到Activity2的信息后，调用Instrumentation类的newActivity方法，其内通过ClassLoader创建Activity2**实例**。（37 - 40）

5. 通知Activity2去performCreate

Binder里有很多线程在跑。fork会把进程里面当前线程复制过去，当线程里某个资源被其他资源锁住时，当fork后线程信息丢失了（fork原理），最后没有开锁的钥匙了导致死锁。socket会把其他线程停掉，fork后是干净的

[Activity的启动流程这一篇够了](https://www.jianshu.com/p/d7364591f1d1)

# 日志采集框架

- 日志采集
- 日志存储
- 日志上报

代码层日志、用户行为日志、网络日志、崩溃日志、H5日志

[[Logan：美团开源移动端基础日志库](https://tech.meituan.com/2018/10/11/logan-open-source.html)](https://tech.meituan.com/2018/10/11/logan-open-source.html)

[[美团移动端基础日志库——Logan](https://tech.meituan.com/2018/02/11/logan.html)](https://tech.meituan.com/2018/02/11/logan.html)

[关于Android日志监控设计的一些想法](https://xybean.github.io/2019/05/04/%E5%85%B3%E4%BA%8EAndroid%E6%97%A5%E5%BF%97%E7%9B%91%E6%8E%A7%E8%AE%BE%E8%AE%A1%E7%9A%84%E4%B8%80%E4%BA%9B%E6%83%B3%E6%B3%95/)



**参考面试题**

[2020 最新 - 今日头条 Android 面试题及答案 (已拿到 offer)](https://www.jianshu.com/p/5e5908ab3ea9)







