> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/cvORMGjd-qfPM_pqozirdA)

在 Android 应用的逆向工程与安全对抗中，**反调试技术**一直是开发者和逆向分析人员之间的博弈重点。应用开发者通过各种手段检测调试器或 Hook 工具（如 Frida、Xposed）的存在，而逆向工程师则会尝试绕过这些检测，以便进行动态调试、注入或内存分析。

其中，一类常见的检测方式是通过**线程检测**来完成的。本文将从线程创建的流程入手，逐步剖析相关原理，并介绍如何在攻防对抗中通过 hook 技术应对线程检测。

反调试检测中的 “线程手段”
--------------

很多安全检测逻辑（例如定时检查调试器状态、扫描内存、监控可疑模块）都会在**新线程**中执行。这么做的原因是：

*   • 保证检测逻辑与主线程解耦，不影响应用正常运行；
    
*   • 检测逻辑能周期性地执行，提高检测稳定性；
    
*   • 线程隐藏在大量业务线程中，增加逆向分析难度。
    

因此，如果我们能在 **新线程启动阶段** 进行拦截，就有机会在第一时间识别并绕过这些检测。

pthread_create：第一个想到的 Hook 点
----------------------------

在 Linux / Android 系统中，创建线程的核心 API 是 `pthread_create`。很多人会想到直接 hook 这个函数，来拦截所有新线程的创建。

然而，`pthread_create` 本身过于显眼，安全厂商和应用开发者往往会**重点检测这个函数是否被篡改**。如果直接 hook，很容易反被检测出来，导致应用直接崩溃或进入防御模式。

因此，**不要直接 hook****`pthread_create`**，我们需要更隐蔽的切入点。

深入分析：线程启动流程
-----------

线程的实际启动流程并不是简单地停留在 `pthread_create`，其调用链大致如下：

```
pthread_create -> clone -> 新线程 -> __start_thread -> __pthread_start -> 执行用户代码

```

关键点有两个：

*   • **`__start_thread`**：这是新线程的入口函数，负责初始化线程环境，并调用 `__pthread_start`。
    
*   • **`__pthread_start`**：最终在新线程环境中执行用户传入的函数（即用户代码）。
    

也就是说，线程的真正 “起跑线” 在 `__start_thread` 与 `__pthread_start` 之间。

IDA 分析 libc
-----------

要更清楚地理解这两个函数，我们可以导出 Android 系统的 `libc.so`，并用 IDA 分析：

```
adb pull /system/lib64/libc.so

```

在 IDA 中找到 `__start_thread` 和 `__pthread_start` 的实现，可以清晰看到它们如何从参数中取出用户代码的入口地址，并最终跳转执行。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/FMiaMMfBpPgBWFqgVUeawE6tCFDsKOyz9VOu8eyWeYojibJMRjnjKEVODVjlWCuwsWkGWdT5LFSBtP5DtFhtRT7g/640?wx_fmt=png&from=appmsg&watermark=1)

例如，在 `__pthread_start` 的汇编中：

```
MOV             X19, X0   ; X0 是 __pthread_start 的第 0 个参数
......
LDP             X8, X0, [X19,#0x60]
BLR             X8        ; 调用用户代码

```

其中：

*   • `X8` 存放用户代码的地址；
    
*   • `X0` 是传给用户代码的第一个参数。
    

这意味着我们只要在 `__start_thread` 或 `__pthread_start` 处 hook，就能拿到新线程的用户代码入口地址。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/FMiaMMfBpPgBWFqgVUeawE6tCFDsKOyz9MjyVVY1HnQtABm0mT43JOQKdbyVW6ZQ7X2ibqJhepa9UHatKBOq8EJw/640?wx_fmt=png&from=appmsg&watermark=1)

如何解析用户线程地址
----------

`__start_thread` 与 `__pthread_start` 都可以作为 hook 点。

*   • `__start_thread` 会将参数直接透传给 `__pthread_start`；
    
*   • 参数中包含了用户代码地址，只需从偏移 `0x60` 处读取即可。
    

示例（Frida 伪代码）：

```
thread.add(0x60).readPointer();  // 获取用户代码地址

```

这样，我们就能知道新线程真正要执行的函数位置，从而决定是否放行。

Hook 示例：拦截线程初始化
---------------

以下示例展示了如何在 Frida 中 hook `__pthread_start`：

```
function hook_start_thread() {
    var libc = Process.findModuleByName("libc.so");
    var addr_start_thread = libc.base.add(0xCB5D8); // __pthread_start 偏移
    var original_start_thread = new NativeFunction(addr_start_thread, "void", ["pointer"]);

    Interceptor.replace(addr_start_thread, new NativeCallback(function (thread) {
        var user_func_addr = thread.add(0x60).readPointer();
        var nice_addr = addr2nice(user_func_addr);

        console.log("thread start: " + nice_addr);

        if (nice_addr.includes("libfrida.so")) { 
            // 检查是否为 Frida 或其他恶意模块创建的线程
            console.log("frida check thread detected");
        } else {
            // 正常线程继续执行
            original_start_thread(thread);
        }
    }, "void", ["pointer"]));
}
hook_start_thread();

```

这个 Hook 做了几件事：

1.  1. **拦截新线程启动**；
    
2.  2. **提取用户代码地址**；
    
3.  3. **检查创建线程的模块**（如 Frida）；
    
4.  4. **决定是否放行原始函数**。
    

本文从线程角度切入，介绍了反调试检测中的一种常见手段，并深入分析了线程启动流程中隐藏的关键函数 `__start_thread` 和 `__pthread_start`。相比直接 Hook `pthread_create`，选择更深层的入口能有效降低被检测的风险。通过示例代码，我们展示了如何在 Frida 中拦截线程初始化、解析用户代码入口，并在攻防对抗中选择性放行线程。对于逆向工程师来说，掌握这种方法不仅能帮助绕过反调试，还能在分析复杂应用时更快抓住重点逻辑。而从防御角度来看，理解攻击者的手段也有助于设计更加稳固的反调试机制。攻防双方的持续较量，正是推动移动安全技术不断进化的动力。

学习资源
====

立即关注【二进制磨剑】公众号

  

👉👉👉[【IDA 技巧合集】](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI1Mjk2MTM1OQ==&action=getalbum&album_id=3353659750835535873#wechat_redirect)👈👈👈

👉👉👉[【Github 安全项目合集】](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI1Mjk2MTM1OQ==&action=getalbum&album_id=3257219530066477058#wechat_redirect)👈👈👈

  

[![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/FMiaMMfBpPgAHcgEpVwNlwbjBT4E0PMAEaJrrdB5hsn6LWIqIBWP1YYNyyc3ISBVwjRlkQJB2ALvx5plHg4PHmA/640?wx_fmt=jpeg&from=appmsg&watermark=1)](https://mp.weixin.qq.com/s?__biz=MzI1Mjk2MTM1OQ==&mid=2247485678&idx=1&sn=2c4735f06b30c42926d0f12f3991d0c7&scene=21#wechat_redirect)