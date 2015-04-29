---
layout: post
title:  "通过Accessibility自动安装与卸载应用"
date:   2015-04-27
categories: Accessibility install uninstall
tags: Accessibility install uninstall
---


Android提供Accessibility为了让用户更容易的操作手机，例如针对与视力障碍的用户提供语音功能，当开锁屏或者其他点击操作能提供语音提示与阅读屏幕内容等。API level 4已经添加但是功能比较简单到API level 14才提供更强大的支持，包括文字转语音，触觉反馈，手势操作，轨迹球和手柄操作。

# 一、 安装与自动安装流程介绍 #

下面通过应用安装过程举例，首先回忆手动安装应用的场景：

1. 软件下载完成后，在页面中点击“安装”按钮
2. 先跳转到系统-设置-安装页面显示“安装”按钮
3. 点击“安装”按钮后出现权限页面显示“下一步”或者“取消”按钮
4. 点击“下一步”当应用安装完成后出现应用安装完成页面，出现“打开”、“完成”按钮。
5. 点击“打开”按钮
6. 进入安装的应用 

再来看下通过Accessibility进行自动安装的场景：

1. 与上面第1步相同与上面相同，后续步骤都有Accessibility接管
2. 通过Accessibility执行“安装”、“下一步”、“打开”按钮进行三次操作完成整个应用安装过程。
3. 与上面第6不相同


# 二、 自动安装卸载例子 #

1. AndroidManifest.xml中注册Service与声明权限

		<manifest>
		  ...
		  <uses-permission ... />
		  ...
		  <application>
		    ...
		    <service android:name=".MyAccessibilityService"
		        android:label="@string/accessibility_service_label"
		        android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
		      <intent-filter>
		        <action android:name="android.accessibilityservice.AccessibilityService" />
		      </intent-filter>
		    </service>
		    <uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE" />
		  </application>
		</manifest>

2. MyAccessibilityService接收事件与处理

		public class MyAccessibilityService extends AccessibilityService {
		    @Override
		    public void onAccessibilityEvent(AccessibilityEvent event) {
		        
		        // 判断是否来自设置应用,此处需要做机型兼容
		        if ("com.android.packageinstaller".equals(event.getPackageName())
		                || "com.lenovo.safecenter".equals(event.getPackageName())) {
		            return;
		        }
		        
		        findAndPerformAction("安装");
		        
		        findAndPerformAction("下一步");
		        
		        findAndPerformAction("打开");            
		    }    
		    
		    private void findAndPerformAction(String text) {
		        // 查找当前窗口中包含“安装”文字的按钮
		        List<AccessibilityNodeInfo> nodes = getRootInActiveWindow().findAccessibilityNodeInfosByText(text);
		        for (int i = 0; i < nodes.size(); i++) {
		            AccessibilityNodeInfo node = nodes.get(i);
		            // 执行按钮点击行为
		            node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
		        }
		    }
		}	




以上是Accessibility自动安装于卸载的例子来自[Android Accessibility(辅助功能) --实现Android应用自动安装、卸载](http://blog.csdn.net/androidsecurity/article/details/41890369?utm_source=tuicool)，比较简单但是发现例子有两个问题，附件中已经修复以下问题

1) 部分手机无法自动点击“打开”按钮
在部分手机（例如HTC 手机）上发现安装过程最后一步显示“打开”按钮界面，使用AccessibilityEvent.getSource()[Added in API level 14]获取到的值为空，导致无法获取触发点击“打开”按钮。
通过AccessibilityService.getRootInActiveWindow ()[Added in API level 16] 获取真个窗口的控件对象信息解决此问题。

2) 部分手机自动安装页面无任何反应
例子中判断需要点击的按钮对象为Button时才触发，但是一些手机上按钮是TextView，通过添加TextView判断条件解决。

例子源码[下载地址](/assets/posts/2015-04-28-Accessibility-automatically-install-and-uninstall/AccessibilityDemo.rar)


# 三、 Accessibility通知策略 #
1) 是否可以同时存在多个Accessibility支持自动安装与卸载的应用？
可以共存，因为类似与广播机制，主要是注册AccessibilityService都能获取到通知信息。

2) 多个支持Accessibility的应用，获取通知的顺序？
与安装应用的先后顺序无关，主要取决于用户在设置页面开启服务的顺序。先开启的服务优于后开启的服务先得到通知信息。

3) 如何独占事件通知？

AccessibilityService可以通过以下方式设置反馈事件类型

        AccessibilityServiceInfo serviceInfo = new AccessibilityServiceInfo();
        // 设置反馈类型
        serviceInfo.feedbackType = AccessibilityServiceInfo.FEEDBACK_HAPTIC;
        AccessibilityService.setServiceInfo(serviceInfo);
                   
反馈事件类型支持6中类型包括：FEEDBACK_AUDIBLE、FEEDBACK_GENERIC、FEEDBACK_HAPTIC、FEEDBACK_SPOKEN、FEEDBACK_VISUAL、FEEDBACK_BRAILLE。默认是FEEDBACK_GENERIC通用类型，如果两个应用都设置此类型都可以接受到事件通知。但是如果两个应用设置的feedbackType相同并且不是FEEDBACK_GENERIC类型（例如两个应用都设置的FEEDBACK_HAPTIC触觉类型反馈），仅有一个应用会获取到事件通知，所以这种情况下仅第一个被用户打开服务的应用才能接受到事件通知（即满足疑问2的条件）。


# 四、模拟用户点击不同实现方案比较 #
1. Accessibility 缺点：API Level 限制仅14、16以上才可使用，需要用户手动开启服务，需要通过文本或viewid查找点击元素。优点速度快
2. Hierarch Viewer 缺点：手机必须root。如果感兴趣可以查看以下系列文章，移动自动测试工具通常都是使用这种方式。
3. UIAutoViewer 缺点：不支持动态的页面布局。


# 五、参考资料 #

1.涉及到的类文档

[AccessibilityService](http://developer.android.com/reference/android/accessibilityservice/AccessibilityService.html)（包含Event types 与 Feedback types）、[AccessibilityEvent](http://developer.android.com/reference/android/view/accessibility/AccessibilityEvent.html)、[AccessibilityServiceInfo](http://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo.html)


2.[Accessibility官方介绍文档](http://developer.android.com/guide/topics/ui/accessibility/index.html)

3.[微信抢红包（比较有意思的Demo）](https://github.com/waylife/RedEnvelopeAssistant)

4.Hierarchy View相关资料

[Android工具HierarchyViewer 代码导读(1) -- 功能实现演示](http://www.cnblogs.com/vowei/archive/2012/07/30/2614353.html)
  
[Android工具HierarchyViewer 代码导读(2) -- 建立Eclipse调试环境](http://www.cnblogs.com/vowei/archive/2012/08/03/2618753.html)

[Android工具HierarchyViewer 代码导读(3) -- 后台代码](http://www.cnblogs.com/vowei/archive/2012/08/08/2627614.html)

[Android工具HierarchyViewer代码导读(4) -- 前台代码](http://www.cnblogs.com/vowei/archive/2012/08/22/2650722.html)
