

Hook 直译过来就是“钩子”的意思，是指截获进程对某个 API 函数的调用，使得 API 的
执行流程转向我们实现的代码片段，从而实现我们所需要得功能，这里的功能可以是监控、
修复系统漏洞，也可以是劫持或者其他恶意行为。





[字节跳动开源 Android PLT hook 方案 bhook](https://zhuanlan.zhihu.com/p/401547387)

[Android 无所不能的 hook，让应用不再崩溃](https://juejin.cn/post/7034178205728636941)

爱奇艺开源的的xHook

Lancet 饿了么

因为最近工作需要用到[AOP](https://so.csdn.net/so/search?q=AOP&spm=1001.2101.3001.7020)技术，如是在网上搜索已经有的AOP框架，找到了[lancet](https://github.com/eleme/lancet)、[booster](https://github.com/didi/booster)和[ByteX](https://github.com/bytedance/ByteX)。

其中lancet比较符合我的要求，但是使用中发现了2个问题所以放弃了。

1、lancet是基于ASM5开发的，ASM5版本过低
