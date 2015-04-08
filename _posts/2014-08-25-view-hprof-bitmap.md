---
layout: post
title:  "MAT内存分析之-查看android内存中的bitmap"
date:   2014-08-25
categories: MAT
tags: hprof MAT android bitmap
---

# 1.MAT分析hprof文件 #

这里假设已经知道如何dump hprof，以及如何使用mat查看内存情况。

# 2.安装GIMP #

下载地址 [http://www.gimp.org/](http://www.gimp.org/)

# 3.MAT中把图片内存数据存储到文件中 #

- 找到要分析的Bitmap对象，查看右键点击他的mBuffer属性

	![](/assets/posts/2014-08-25-view-hprof-bitmap/bitmap-buffer.png)

- 在右键菜单中选择 Copy -> Save value to file

	把bitmap对应的二进制数据存储到磁盘上，比如命名为 bitmap.data  ， 不要命名为bmp等已知的图片格式后缀。

- 查看图片的宽高

	在 mat 中 bitmap的属性中可以看到
	![](/assets/posts/2014-08-25-view-hprof-bitmap/bitmap-property.png)

- 使用GIMP查看图片

	使用 GIMG 打开刚才存储的 bitmap.data。  
	图片类型选择 RGB Alpha ,  
	宽度和高度，填入上一个步骤获取到的宽高，  
	然后打开。

	![](/assets/posts/2014-08-25-view-hprof-bitmap/gimg.png)