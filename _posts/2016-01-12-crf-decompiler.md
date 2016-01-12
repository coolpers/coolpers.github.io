---
layout: post
title:  "CFR Java Decompiler"
date:   2016-01-12
categories: CRF
tags: CFR Decompiler
---

# java反编译，JAD & CFR #

目前我们开发中大都使用JAD进行java反编译。这个工具已经过于陈旧，最突出的问题就是经常反编译出错。

使用CFR反编译工具能够很好的解决这个问题，并且支持java8，这个工具更活跃。

# CFR  #

附件是Java反编译工具CFR，支持java7，java8的反编译，能解决jd-gui部分代码不能反编译的问题，尤其是匿名类，内部类的一些逻辑。使用方法如下：

    D:\>java -jar cfr_0_110.jar D:\example.jar –outputdir D:\data\example

输入可以是jar，class文件，也可以是在classpath里的类名，其他的高级选项可以参考—help，或者官网说明，最新版本也可以去官网获取，目前最新的是110版本。

官网地址：[http://www.benf.org/other/cfr/](http://www.benf.org/other/cfr/ "http://www.benf.org/other/cfr/")

