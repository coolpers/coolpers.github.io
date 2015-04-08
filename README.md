# http://coolpers.github.io

百度手机助手客户端研发团队空间。

## 新建 blog 注意事项 ##

### 1、blog markdown 命名格式 ###
在`_posts`目录下创建blog内容mardown，命名格式必须为 年-月-日-名称.md, 其中年为4位，月日为2位，比如 `2015-04-07-compile-MAT.md`，这里最好不要用中文。使用中划线分割。

### 2、blog markdown 内容格式 ###

blog 内容分为 header区和content区，格式如下。

	---
	layout: post
	title:  "Compile MAT Source Code"
	date:   2015-04-07
	categories: MAT
	tags: MAT
	---
	# 1.MAT源代码下载 #
	MAT 官网介绍 https://eclipse.org/mat

- layout: post 不用修改，意思为此md使用 _layouts/post.html 进行包装显示
- title: "This is Title"  就是文章title，显示在文章html中。
- date: 文章发布的真正时间，用于站点显示。
- categories: 表示文章的 category，可以为多个，用空格翻开。建议几组相同的文章使用相同的 category，方便站点使用category进行分类查找。
- tags: 同 category。

### 3、blog中链接的图片或者二进制文件 ###

- 文章中链接的图片或者其他二进制文件，存放在 assets/posts/xxxx-xx-xx-title  目录下，也就是自己某个文章的二进制内容需要在 assets/posts 目录下创建一个属于自己的独立目录，所有文档放到该目录下。
- 图片的引用，`![title](/assets/posts/xxxx-xx-xx-title/test.png)`
- 下载链接的引用，`[下载地址](/assets/posts/2015-04-07-compile-MAT/MAT_R_1.4.0.zip)`




	
