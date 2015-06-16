---
layout: post
title:  "LeakCanary原理分析"
date:   2015-06-04
categories: LeakCanary|MAT
tags: LeakCanary MAT
---
#### 导语：
>提到Java语言的特点，无论是教科书还是程序员一般都会罗列出面向对象、可移植性及安全等特点。但如果你是一位刚从C/C++转到Java的程序员，对Java语言的特性除了面向对象之外，最外直接的应当是在Java虚拟机（JVM）在内存管理方面给我们变成带来的便利。JVM的这一大特性使Java程序员从繁琐的内存管理工作中得到了一定解放，但是JVM的这个特点的实现也是有代价的，并且它也并非万能。因此如果一个编程习惯不好的Java程序员如果完全将内存回收寄希望于JVM，那么OOM（Out Of Memory）就已经悄悄潜伏在了他的程序之中。

>Android应用基于Java实现，因此它也将Java的优缺点继承了过来。相对来说，移动设备对于内存问题更为敏感，程序在申请一定的内存但又没有及时得到释放后就很容易发生OOM而导致crash。因此Android程序员开发过程中一般都会定时排查自己程序中可能出现的这些雷点，尽可能地避免因为crash问题而影响用户体验。


## 1.LeakCanary简介 ##
目前Java程序最常用的内存分析工具应该是MAT（Memory Analyzer Tool），它是一个Eclipse插件，同时也有单独的RCP客户端，也可以通过官网的SVN下载到它的源码（具体见另一篇《compile-MAT》）并编译成jar包。LeakCanary本质上就是一个基于MAT进行Android应用程序内存泄漏自动化检测的的开源工具，通过集成这个工具代码到自己的Android工程当中就能够在程序调试开发过程中通过一个漂亮的界面（如下图）随时发现和定位内存泄漏问题，而不用每次在开发流程中都抽出专人来进行内存泄漏问题检测，极大地方便了Android应用程序的开发。

![LeakCanary_result](/assets/posts/2015-06-16-LeakCanary-Brief/LeakCanary_result.png)

总的来说，LeakCanary有如下几个明显优点：

*	针对Android Activity组件完全自动化的内存泄漏检查。
*	可定制一些行为（dump文件和leaktrace对象的数量、自定义例外、分析结果的自定义处理等）。
*	集成到自己工程并使用的成本很低。
*	友好的界面展示和通知。

假如你现在想集成LeakCanary到自己的工程中，那么你只需要做以下工作：
1. 导入leakcanary的jar包到自己工程（下载链接：![leakcanary.zip](/assets/posts/2015-06-16-LeakCanary-Brief/leakcanary.zip)）
2. 在4.0以上，只需要在工程的Application的onCreate函数中按照如下的方式加入一行代码：


    public class ExampleApplication extends Application {
          @Override
          public void onCreate() {
            super.onCreate();
            LeakCanary.install(this);
          }
    }

　　4.0以下在需要进行内存泄漏监控的Activity的onDestroy方法中按如下加入代码：

    protected void onDestroy() {
        super.onDestroy();
        // start watch
        HeapDump.Listener heapDumpListener =
        new ServiceHeapDumpListener(this, listenerServiceClass);
        DebuggerControl debuggerControl = new AndroidDebuggerControl();
        AndroidHeapDumper heapDumper = new AndroidHeapDumper();
        heapDumper.cleanup();
        ExcludedRefs excludedRefs = AndroidExcludedRefs.createAppDefaults().build();
        RefWatcher refWatcher = new RefWatcher(new AndroidWatchExecutor(), debuggerControl, GcTrigger.DEFAULT,
        heapDumper, heapDumpListener, excludedRefs);
    }
　　第二种情况下，在有多个Activity需要检测的情况看起来稍显繁琐，实际上可以用以上方法实现一个基类Activity，之后需要内存泄漏检测的Activity直接继承这个基类Activity就不需要每次都重复处理oonDestroy方法了。并且以上代码只作为示例，实际上每次watch的时候并不需要重新new一个RefWatcher对象，因为这个对象是可以重复使用的。

完成了以上两个步骤后，LeakCanary就可以为你的工程服务了，这之中需要我们自己处理的工作很少，相比较我们自己手工用MAT进行内存泄漏检测而言，确实方便了很多。
## 2.LeakCanary原理分析 ##
这么强大的工具，它是如何实现的呢，引用[LeakCanary中文使用说明](http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)，它的基本工作原理如下：

1. RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。
2. 然后在后台线程检查引用是否被清除，如果没有，调用GC。
3. 如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 .hprof 文件中。
4. 在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer 使用HAHA 解析这个文件。
5. 得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄漏。
6. HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄漏。如果是的话，建立导致泄漏的引用链。
7. 引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来。

但事实上一切并没那么简单，LeakCanary的设计者在实现的时候实际上为我们考虑了很多细节。可以通过源码分析来走一遍一次内存泄漏检查的流程。
在一个Activity生命周期结束调用oonDestroy方法的时候会触发LeakCanary进行一次内存泄漏检查，LeakCanary开始进行检查的入口函数实际上是RefWatcher类的，watch方法，其源码如下：


    public void watch(Object watchedReference, String referenceName) {

      ...

      String key = UUID.randomUUID().toString();
      retainedKeys.add(key);
      final KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, queue);

      watchExecutor.execute(new Runnable() {
          @Override
          public void run() {
              ensureGone(reference, watchStartNanoTime);
          }
      });
    }
这个函数做的主要工作就是为需要进行泄漏检查的Activity创建一个带有唯一key标志的弱引用，并将这个弱引用key保存至retainedKeys中，然后将后面的工作交给watchExecutor来执行。这个watchExecutor在LeakCanary中是AndroidWatchExecutor的实例，调用它的execute方法实际上就是向主线程的消息队列中插入了一个IdleHandler消息，这个消息只有在对应的消息队列为空的时候才会去执行，因此RefWatcher的watch方法就保证了在主线程空闲的时候才会去执行ensureGone方法，防止因为内存泄漏检查任务而严重影响应用的正常执行。ensureGone的主要源码如下:

    void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
        ...

        removeWeaklyReachableReferences();
        if (gone(reference) || debuggerControl.isDebuggerAttached()) {
            return;
        }
        gcTrigger.runGc();      // 手动执行一次gc
        removeWeaklyReachableReferences();
        if (!gone(reference)) {

            long startDumpHeap = System.nanoTime();
            long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

            File heapDumpFile = heapDumper.dumpHeap();
            if (heapDumpFile == null) {
                // Could not dump the heap, abort.
                Log.d(TAG, "Could not dump the heap, abort.");
                return;
            }
            long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

            heapdumpListener.analyze(new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs,
                    watchDurationMs, gcDurationMs, heapDumpDurationMs));
        }
      }
因为这个方法是在主线程中执行的，因此一般执行到这个方法中的时候之前被destroy的那个Activity的资源应该被JVM回收了，因此这个方法首先调用removeWeaklyReachableReferences方法来将引用队列中存在的弱引用从retainedKeys中删除掉，这样，retainedKeys中保留的就是还没有被回收对象的弱引用key。之后再用gone方法来判断我们需要检查的Activity的弱引用是否在retainedKeys中，如果没有，说明这个Activity对象已经被回收，检查结束。否则，LeakCanary主动触发一次gc，再进行以上两个步骤，如果发现这个Activity还没有被回收，就认为这个Activity很有可能泄漏了，并dump出当前的内存文件供之后进行分析。

之后的工作就是对内存文件进行分析，由于这个过程比较耗时，因此最终会把这个工作交给运行在另外一个进程中的HeapAnalyzerService来执行。HeapAnalyzerService通过调用HeapAnalyzer的checkForLeak方法进行内存分析，其源码如下：

    public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {

        ...

        ISnapshot snapshot = null;
        try {
          snapshot = openSnapshot(heapDumpFile);  

          IObject leakingRef = findLeakingReference(referenceKey, snapshot);

          // False alarm, weak reference was cleared in between key check and heap dump.
          if (leakingRef == null) {
            return noLeak(since(analysisStartNanoTime));
          }

          String className = leakingRef.getClazz().getName();

          AnalysisResult result =
              findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, true);

          if (!result.leakFound) {
            result = findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, false);
          }

          return result;
        } catch (SnapshotException e) {
          return failure(e, since(analysisStartNanoTime));
        } finally {
          cleanup(heapDumpFile, snapshot);
        }
    }

这个方法进行的第一步就是利用HAHA将之前dump出来的内存文件解析成Snapshot对象，其中调用到的方法包括SnapshotFactory的parse和HprofIndexBuilder的fill方法。解析得到的Snapshot对象直观上和我们使用MAT进行内存分析时候罗列出内存中各个对象的结构很相似，它通过对象之间的引用链关系构成了一棵树，我们可以在这个树种查询到各个对象的信息，包括它的Class对象信息、内存地址、持有的引用及被持有的引用关系等。到了这一阶段，HAHA的任务就算完成，之后LeakCanary就需要在Snapshot中找到一条有效的到被泄漏对象之间的引用路径。首先它调用findLeakTrace方法来从Snapshot中找到被泄漏对象，源码如下：


    private IObject findLeakingReference(String key, ISnapshot snapshot) throws SnapshotException {

        Collection<IClass> refClasses =
            snapshot.getClassesByName(KeyedWeakReference.class.getName(), false);

        if (refClasses.size() != 1) {
          throw new IllegalStateException(
              "Expecting one class for " + KeyedWeakReference.class.getName() + " in " + refClasses);
        }

        IClass refClass = refClasses.iterator().next();

        int[] weakRefInstanceIds = refClass.getObjectIds();

        for (int weakRefInstanceId : weakRefInstanceIds) {
          IObject weakRef = snapshot.getObject(weakRefInstanceId);
          String keyCandidate =
              PrettyPrinter.objectAsString((IObject) weakRef.resolveValue("key"), 100);  
          if (keyCandidate.equals(key)) {  // 匹配key
            return (IObject) weakRef.resolveValue("referent");  // 定位泄漏对象
          }
        }
        throw new IllegalStateException("Could not find weak reference with key " + key);
      }

为了能够准确找到被泄漏对象，LeakCanary通过被泄漏对象的弱引用来在Snapshot中定位它。因为，如果一个对象被泄漏，一定也可以在内存中找到这个对象的弱引用，再通过弱引用对象的referent就可以直接定位被泄漏对象。
下一步的工作就是找到一条有效的到被泄漏对象的最短的引用，这通过findLeakTrace来实现，实际上寻找最短路径的逻辑主要是封装在PathsFromGCRootsComputerImpl这个类的getNextShortestPath和processCurrentReferrefs这两个方法当中，其源码如下：

    public int[] getNextShortestPath() throws SnapshotException {
          switch (state) {
            case 0: // INITIAL
            {

              ...
            }
            case 1: // FINAL
              return null;

            case 2: // PROCESSING GC ROOT
            {
              ...
            }
            case 3: // NORMAL PROCESSING
            {
              int[] res;

              // finish processing the current entry
              if (currentReferrers != null) {
                res = processCurrentReferrefs(lastReadReferrer + 1);
                if (res != null) return res;
              }

              // Continue with the FIFO
              while (fifo.size() > 0) {
                currentPath = fifo.getFirst();
                fifo.removeFirst();
                currentId = currentPath.getIndex();
                currentReferrers = inboundIndex.get(currentId);

                if (currentReferrers != null) {
                  res = processCurrentReferrefs(0);
                  if (res != null) return res;
                }
              }
              return null;
            }

            default:
              ...
          }
        }


        private int[] processCurrentReferrefs(int fromIndex) throws SnapshotException {
          GCRootInfo[] rootInfo = null;
          for (int i = fromIndex; i < currentReferrers.length; i++) {
            ...
          }
          for (int referrer : currentReferrers) {
            if (referrer >= 0 && !visited.get(referrer) && !roots.containsKey(referrer)) {
              if (excludeMap == null) {
                fifo.add(new Path(referrer, currentPath));
                visited.set(referrer);
              } else {
                if (!refersOnlyThroughExcluded(referrer, currentId)) {
                  fifo.add(new Path(referrer, currentPath));
                  visited.set(referrer);
                }
              }
            }
          }
          return null;
        }
      }

为了是逻辑更清晰，在这里省略了对GCRoot的处理。这个类将整个内存映像信息抽象成了一个以GCRoot为根的树，getNextShortestPath的状态3是对一般节点的处理，由于之前已经定位了被泄漏的对象在这棵树中的位置，为了找到一条到GCRoot最短的路径，PathsFromGCRootsComputerImpl采用的方法是类似于广度优先的搜索策略，在getNextShortestPath中从被泄漏的对象开始，调用一次processCurrentReferrefs将持有它引用的节点（父节点），加入到一个FIFO队列中，然后依次再调用getNextShortestPath和processCurrentReferrefs来从FIFO中取节点及将这个节点的父节点再加入FIFO队列中，一层一层向上寻找，哪条路径最先到达GCRoot就表示它应该是一条最短路径。由于FIFO保存了查询信息，因此如果要找次最短路径只需要再调用一次getNextShortestPath触发下一次查找即可。
至此，主要的工作就完成了，后面就是调用buildLeakTrace构建查询结果，这个过程相对简单，仅仅是将之前查找的最短路径转换成最后需要显示的LeakTrace对象，这个对象中包括了一个由路径上各个节点LeakTraceElement组成的链表，代表了检查到的最短泄漏路径。最后一个步骤就是将这些结果封装成AnalysisResult对象然后交给DisplayLeakService进行处理。这个service主要的工作是将检查结果写入文件，以便之后能够直接看到最近几次内存泄露的分析结果，同时以notification的方式通知用户检测到了一次内存泄漏。使用者还可以继承这个service类来并实现afterDefaultHandling来自定义对检查结果的处理，比如将结果上传刚到服务器等。
以上就是对LeakCanary源码的分析，中间省略了一些细节处理的说明，但不得不提的是LeakCanary支持自定义泄漏豁对象ExcludedRefs的集合，这些豁免对象一般都是一些已知的系统泄漏问题或者自己工程中已知但又需要被排除在检查之外的泄漏问题构成的。LeakCanary在findLeakTrace方法中如果发现这个集合中的对象存在于泄漏路径上，就会排除掉这条泄漏路径并尝试寻找下一条。
