系统服务在触发ANR后，会发送一个SIGQUIT信号到应用进程来触发dump traces

监控方案有三种：

1. 监听anr目录的变化
  - 使用FielProvider监听/data/anr/traces.txt文件变化，并捕获现场进行上报。
2. 主线程超时监测
 - 开启一个子线程定期post一个message到主线程，隔一段时间检测该message是否被消费掉，如果没有被处理，则说明主线程被卡住，可能发生ANR,再通过系统服务获取当前进程的错误信息，判断是否有ANR发生。 但这个会存在大量漏报的情况，并且轮询的方案性能不佳
3. 监听 SIGQUIT 信号
 - 系统服务在触发ANR后，会发送一个SIGQUIT信号到应用进程来触发dump traces，在应用侧我们可以监听SIGQUIT信号来判断是否发生了ANR。为了排除其他进程的ANR导致的误报，需要再通过系统服务获取当前进程的错误信息，进一步过滤。

## 叨一叨ANR

### ANR

如果Android应用界面线程处于阻塞状态过长，会触发”应用无响应"(ANR)错。ANR由消息处理机制保证，在系统层实现了一套机制来发现ANR，核心原理是消息调度和超时处理。

出现以下问题时，系统都会针对你的应用触发ANR：

1. 当你的Activity处于前台，你的应用在5秒内没有响应事件或BroadReceiver
2. 虽然前台没有Activity，但你的BroadcastReceive用了很长的时间仍然没有执行完毕

在主体实现方面，所有与ANR有关的消息，都会经过系统进程的调度，同时系统进程设计了不同的超时限制来跟踪消息的处理，在消息处理不当的时候，超时限制就会起作用，并且收集一些系统状态，比如CPU/IO使用情况、进程函数调用栈，并且报告用户有进程无响应了(ANR对话框)。

那系统又是如何检测超时的，这里就来看下如何对Service进行超时检测：

在`onCreate`生命周期开始执行前，启动超时监测，如果在指定的时间onCreate没有执行完毕（该该方法中执行耗时任务），就会调用`ActiveServices.serviceTimeout()`方法报告ANR；如果在指定的时间内`onCreate`执行完毕，那么就会调用`ActivityManagerService.serviceDoneExecutingLocked()`方法移除`SERVICE_TIMEOUT_MSG`消息，说明Service.onCreate方法没有发生ANR，Service是由AMS调度，利用Handler和Looper，设计了一个TIMEOUT消息交由AMS线程来处理，整个超时机制的实现都是在Java层，下面就详细来看下：

在开启Service的过程中，会在`ActiviteService`的`realStartServiceLocked`方法中进行设置超时检测：

```java
 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        ......
          //为了设置ANR超时，在启动之前进行ANR检测
        bumpServiceExecutingLocked(r, execInFg, "create");
   .....
     //最终会启动onCreate方法
    app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
   ......
     //绑定过程中
      requestServiceBindingsLocked(r, execInFg);
 }
```

后面`bumpServiceExecutingLocked`方法又执行了`scheduleServiceTimeoutLocked`，这里会发送一个`SERVICE_TIMEOUT_MSG`消息，并根据前台后台进行区分设置，前台20s，后台200s。

```java
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
  //
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
  //这里会发送SERVICE_TIMEOUT_MSG消息
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
```

这里进行了设置了，那就会有个地方要进行移除的操作。在serviceDoneExecutingLocked中会remove该SERVICE_TIMEOUT_MSG消息：

```java
void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying, boolean finishing) {
  ......
    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
    
}
```

在超时后仍然没移除`SERVICE_TIMEOUT_MSG`消息，就会执行`serviceTimeOut`方法:

```java
void serviceTimeout(ProcessRecord proc) {
  ......
         long now = SystemClock.uptimeMillis();
            final long maxTime =  now -
                    (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;
  //寻找运行超时的service
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }
  //判断执行service超时的进程是否在最近运行的列表里
            if (timeout != null && mAm.mProcessList.mLruProcesses.contains(proc)) {
                Slog.w(TAG, "Timeout executing service: " + timeout);
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortInstanceName;
            } else {
              //不在进行移除
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        if (anrMessage != null) {
          //当有超时的消息的时候，执行appNotResponding，报告ANR
            mAm.mAnrHelper.appNotResponding(proc, anrMessage);
        }
    }
```

以上就是Service超时监测的整体流程。



### ANR如何治理

在对ANR的治理中，经常会看到将卡顿阈值设为5s来监控ANR，这也是



NativePollOnce 场景



**参考**

[黑科技！让Native Crash 与ANR无处发泄！](https://juejin.cn/post/7114181318644072479)

[今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://juejin.cn/post/6940061649348853796#heading-6)

[看完这篇 Android ANR 分析，就可以和面试官装逼了！](https://mp.weixin.qq.com/s/4w202K0WnNrazmEHd6grQA)

[Android修炼系列（32），你理解的 ANR 监控可能一直是错的](https://juejin.cn/post/7077710481837785096)

[关于闲鱼的ANR治理，我有几条心得...](https://juejin.cn/post/6992831566627995685#heading-7)
