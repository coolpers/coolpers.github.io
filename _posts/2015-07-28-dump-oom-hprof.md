---
layout: post
title:  "Android monkey OOM 问题分析"
date:   2015-07-28
categories: oom hprof
tags: oom hprof
---

### 遇到的问题 ###

android app 经常会遇到 OOM 问题，有些 OOM比较容易解决，比如常见的Activity泄漏导致的 OOM。但是有些OOM自己不是很容易复现，monkey 可以复现。这种情况下我们就需要 monkey OOM情况下的 hprof 文件 来分析。

### dump hprof 的几种方法 ###

- 使用 ddms 可以手工dump，这种只能用于手工验证。
- adb shell am dumpheap <pid> <output-file-name> 这个可以实现自动化，参考 源码 /dalvik/docs/heap-profiling.html 中的说明。但是有个问题，此方法对于已经crash 的进程，无法dump。此方法只对 debugable app 有效。
- 参考 ddms 的实现方法，基于 ddmlib 参考 ddms 源码可以 开发一个命令行工具，也很好用。这种情况用于 monkey 结束后 dump，很不错的方法。且 crash 状态写也可以，因为crash状态下的进程也是 running状态，跟am dumpheap 实现方案不一样。此方法只对 debugable app 有效。
- 程序源代码调用 Debug.dumpHprofData 函数输出。


考虑到我们的monkey 是 ignore-crash  ignore-timeout ，也就是发生异常继续执行。所以这种情况下采用 Debug.dumpHprofData 方式是比较合适的。

### Debug.dumpHprofData 方法 ###

- 因为是调试代码，所以只在 debugable 模式下工作，release 代码不生效。
- 注册 UncaughtExceptionHandler ，如果是 OOM 则 dump 内存 到 /sdcard/bdhprof/ 目录，发生一次生成一个，文件格式 packagename_yyyy-mm-dd-HH-mm-ss.hprof,例如 com.baidu.appsearch_2015-07-28-14-08-11.hprof
- Monkey 跑完后，在脚本中添加  adb pull /sdcard/bdhprof/ .  把 hprof 文件 抓取到 本地供分析。然后删除 sdcard中的文件 adb shell rm /sdcard/bdhprof/*

参考代码

	public class MainActivity extends ActionBarActivity {
    
	    ArrayList<byte[]> cache = new ArrayList<byte[]>();
	    
	    private UncaughtExceptionHandler defaultCrashHandler;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        
	        Button button = (Button) findViewById(R.id.button);
	        button.setOnClickListener(new View.OnClickListener() {
	            
	            @Override
	            public void onClick(View v) {
	                
	                defaultCrashHandler = Thread.getDefaultUncaughtExceptionHandler();
	                
	                Thread.setDefaultUncaughtExceptionHandler(new CrashHandler());
	                
					// 模拟 OOM
	                for (;;) {
	                    byte[] c = new byte[1000*1000];
	                    cache.add(c);
	                }
	                
	         
	            }
	        });
	    }
	
    
	    
	    public class CrashHandler implements UncaughtExceptionHandler {
	        
	
	        @Override
	        public void uncaughtException(Thread thread, Throwable ex){
	            
	            if (ex.getClass().equals(OutOfMemoryError.class)) {
	                // 如果是 oom 异常， dump hprof 到 /sdcard/bdhprof/ 目录下
	                File dir = new File(Environment.getExternalStorageDirectory().getPath(), "bdhprof");
	                if (!dir.exists()) {
	                    dir.mkdir();
	                }
	                
	                SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss");
	                String fileName = getPackageName() + "_" + format.format(new Date(System.currentTimeMillis()));
	                
	                
	                File file = new File(dir, fileName + ".hprof");
	                
	                System.gc();
	                
	                try {
	                    Debug.dumpHprofData(file.getAbsolutePath());
	                } catch (IOException e) {
	                    e.printStackTrace();
	                }
	            }
	            
	            // 重新交给系统，抛出dialog 提示异常。
	            defaultCrashHandler.uncaughtException(thread, ex);
	        }
	        
	    }
	}


### 负面影响 ###

dump过程需要时间时间长，此时 monkey 还在继续，所以这时候会产生 anr。 所以分析结果的时候，需要手工过滤掉此类的 anr为正常。
