---
layout: post
title:  "Compile MAT Source Code"
date:   2015-04-07
categories: MAT
tags: MAT
---


# 1.MAT源代码下载 #

MAT 官网介绍 https://eclipse.org/mat/

MAT 开源代码为 svn 仓库，可以通过官网的 `View SVN` 找到下载地址。这里以1.4.0 版本为例：

>  https://dev.eclipse.org/svnroot/tools/org.eclipse.mat/tags/R_1.4.0

使用svn工具下载即可。


也可以从本站下载，[点击开始下载](/assets/posts/2015-04-07-compile-MAT/MAT_R_1.4.0.zip)


# 2.编译环境 #

这里以windows为例。

- 安装 Java 1.6, 高版本比如1.8 导致编译错误。
- 安装 Maven 3.0，不能是3.1 或者 3.2，高版本会导致编译错误。设置环境变量 `M2_HOME` 指向 Maven 安装根目录。

# 3. 预编译prepare_build #

	> cd prepare_build
	> mvn clean install

完成后就在 prepare_build/target 目录下看到依赖的 IBM dtfj（IBM Diagnostic Tool Framework for Java ） 相关文件。


dtfj说明 https://www.ibm.com/developerworks/java/jdk/tools/dtfj.html

# 4. 编译 #
进入 parent 目录，查看 pom.xml ，里边列举了针对不同平台的配置，我们可以对其进行修改，只输出我们关心的平台，比如windows

		<groupId>org.eclipse.tycho</groupId>
			<artifactId>target-platform-configuration</artifactId>
			<version>${tycho-version}</version>
			<configuration>
				<environments>
					<environment>
						<os>win32</os>
						<ws>win32</ws>
						<arch>x86</arch>
					</environment>
					<environment>
						<os>win32</os>
						<ws>win32</ws>
						<arch>x86_64</arch>
					</environment>

编译，直接运行：
	
	> build.bat
	

或者

	> mvn clean
	> mvn install

编译完成后可以在以下目录查找对应的输出。

- org.eclipse.mat.updatesite\target   目录为eclipse mat 插件。
- org.eclipse.mat.product\target\products    目录为mat独立运行软件。