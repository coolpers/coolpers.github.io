---
layout: post
title:  "Android 64位兼容方式运行32位分析"
date:   2015-10-14
categories: android 64bit 32bit 
tags: android 64bit 32bit 
---

关于Android L 64位系统兼容32位应用的实现的简单分析。
 
Android L 的zygote进程的实现不同于之前的版本，除了有zygote进程之外还有zygote64进程。
在init.zygote32_64.rc中有明确指出：

	
	service zygote /system/bin/app_process32 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
	...
	service zygote_secondary /system/bin/app_process64 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
	...


其中app_process32 和app_process64 就是zygote进程的可执行程序，启动后会改名成zygote。

顾名思义，zygote32即app_process32是一个运行在32位的进程，它所连接的库也都是32位的。而zygote64就是运行在64位的进程，它所连接的库都是64位的。

在不考虑有32/64兼容库的情况下，一个进程如果要正确运行，就必须从可执行程序入口开始到所有使用的库都保持32/64位的一致性。

因为zygote进程是所有第三方应用程序的父进程，所以可以认为，如果应用程序是32位的，那没他的父进程也肯定是32位，换句话说，如果需要启动某个32位的应用，那么肯定是通过32位的zygote进程fork出来的。

这个一点可以在ActivityManagerService上得到验证。

ActivityManagerService中startProcessLocked 方法实现启动应用，主要通过Process中的startViaZygote方法，这个方法最终是向相应的zygote进程发出fork的请求
zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);

其中openZygoteSocketIfNeeded(abi)会根据abi的类型，选择不同的zygote的socket监听的端口，在之前的init文件中可以看到

zygote32位监听的端口就是--socket-name=zygote

另外一个就是--socket-name=zygote_secondary

因此可以证实，之前的猜测，即32应用进由32位zygote进程fork出来，64位应用进程由64zygote进程fork出来。

那么之前说的abi参数就是决定应用是32还是64位的关键所在，跟踪这个参数，发现这个参数在ApplicationInfo的primaryCpuAbi中决定，

这个值由PackageManagerService在做scanPackageLI的时候决定，具体这个值的得出有一个公式化的过程，主要就是判断这个apk有没有使用native的库，如果使用了，那就看使用了的是32位的还是64位的，另外还要看系统支持的是32位还是64位的。

> 在64位设备上，如果app的 lib 目录下 存在armeabi，则以32位兼容方式运行。如果存在arm64-v8a 则已64位运行。如果没有任何 so，则 primaryCpuAbi 为空，按照系统的默认配置决定，也就是64位运行。

根据这些因素就可以决定这个apk是应该是32位的还是64位的。
 
以上就是Android L 64位系统兼容32位应用的基本实现过程。

另外记录一点，在源码环境下如果要PREBUILT第三方的so，如果是32位的需要专门标注
 LOCAL_MULTILIB := 32

以此告诉编译系统so位32位，防止编译到64位下去。

来源 http://stackoverflow.com/questions/27712921/how-to-use-32bit-native-libraries-on-64-bit-android-l-platform

> 本文转自 http://blog.csdn.net/louyong0571/article/details/44223481