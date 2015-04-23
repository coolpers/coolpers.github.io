---
layout: post
title:  "mainDexClasses脚本分析"
date:   2015-04-23
categories: MultiDex mainDexClasses
tags: MultiDex mainDexClasses
---

# 1.前言 #
<p style="text-indent:2em;">
前面提到了<a href="https://coolpers.github.io/multidex/2015/04/13/multidex.html">multidex的打包过程</a>，对于哪些类需要打入主Dex中，SDK提供了mainDexClasses脚本来生成文件列表，这两天研究了下该脚本的实现。下面是mainDexClasses的shell脚本,windows下的bat脚本执行步骤一样。
</p>

<pre>
<code>
function makeTempJar ()
{
  local tempDir=/tmp
  if [ ! -e "${tempDir}" ]; then
    tempDir=.
  fi
  local tempfile="${tempDir}/mainDexClasses-$$.tmp.jar"
  if [ -e "${tempfile}" ]; then
    echo "Failed to create temporary file" >2
    exit 6
  fi
  echo "${tempfile}"
}

function cleanTmp ()
{
  if [ -e "${tmpOut}" ] ; then
    rm "${tmpOut}"
  fi
}


# Set up prog to be the path of this script, including following symlinks,
# and set up progdir to be the fully-qualified pathname of its directory.
prog="$0"

while [ -h "${prog}" ]; do
    newProg=`/bin/ls -ld "${prog}"`
    newProg=`expr "${newProg}" : ".* -> \(.*\)$"`
    if expr "x${newProg}" : 'x/' >/dev/null; then
        prog="${newProg}"
    else
        progdir=`dirname "${prog}"`
        prog="${progdir}/${newProg}"
    fi
done
oldwd=`pwd`
progdir=`dirname "${prog}"`
cd "${progdir}"
progdir=`pwd`
prog="${progdir}"/`basename "${prog}"`
cd "${oldwd}"

baserules="${progdir}"/mainDexClasses.rules
if [ ! -r ${baserules} ]; then
    echo `basename "$prog"`": can't find mainDexClasses.rules" 1>&2
    exit 1
fi

jarfile=dx.jar
libdir="$progdir"

if [ ! -r "$libdir/$jarfile" ]; then
    # set dx.jar location for the SDK case
    libdir="$libdir/lib"
fi


if [ ! -r "$libdir/$jarfile" ]; then
    # set dx.jar location for the Android tree case
    libdir=`dirname "$progdir"`/framework
fi

if [ ! -r "$libdir/$jarfile" ]; then
    echo `basename "$prog"`": can't find $jarfile" 1>&2
    exit 1
fi

proguardExec="proguard.sh"
proguard=${PROGUARD_HOME}/bin/${proguardExec}

if [ ! -r "${proguard}" ]; then
  # set proguard location for the SDK case
  proguardBaseDir=`dirname "$progdir"`
  # "${progdir}"/../..
  proguardBaseDir=`dirname "$proguardBaseDir"`
  proguard="${proguardBaseDir}"/tools/proguard/bin/${proguardExec}
fi

if [ ! -r "${proguard}" ]; then
  # set proguard location for the Android tree case
  proguardBaseDir=`dirname "$proguardBaseDir"`
  # "${progdir}"/../../../..
  proguardBaseDir=`dirname "$proguardBaseDir"`
  proguard="${proguardBaseDir}"/external/proguard/bin/${proguardExec}
fi

if [ ! -r "${proguard}" ]; then
    proguard="`which proguard`"
fi

if [ -z "${proguard}" -o ! -r "${proguard}" ]; then
    proguard="`which ${proguardExec}`"
fi

if [ -z "${proguard}" -o ! -r "${proguard}" ]; then
    echo `basename "$prog"`": can't find ${proguardExec}" 1>&2
    exit 1
fi

shrinkedAndroidJar="${SHRINKED_ANDROID_JAR}"
if [ -z "${shrinkedAndroidJar}" ]; then
  shrinkedAndroidJar=shrinkedAndroid.jar
fi

if [ ! -r "${shrinkedAndroidJar}" ]; then
  shrinkedAndroidJar=${libdir}/${shrinkedAndroidJar}
fi

if [ ! -r "${shrinkedAndroidJar}" ]; then
    echo `basename "$prog"`": can't find shrinkedAndroid.jar" 1>&2
    exit 1
fi

if [ "$OSTYPE" = "cygwin" ]; then
    # For Cygwin, convert the jarfile path into native Windows style.
    jarpath=`cygpath -w "$libdir/$jarfile"`
  proguard=`cygpath -w "${proguard}"`
  shrinkedAndroidJar=`cygpath -w "${shrinkedAndroidJar}"`
else
    jarpath="$libdir/$jarfile"
fi

if expr "x$1" : 'x--output' >/dev/null; then
    exec 1>$2
    shift 2
fi

if [ $# -ne 1 ]; then
  echo "Usage : $0 [--output <output file>] <application path>" 1>&2
  exit 2
fi

tmpOut=`makeTempJar`

trap cleanTmp 0

${proguard} -injars ${@} -dontwarn -forceprocessing  -outjars ${tmpOut} \
  -libraryjars "${shrinkedAndroidJar}" -dontoptimize -dontobfuscate -dontpreverify \
  -include "${baserules}" 1>/dev/null || exit 10

java -cp "$jarpath" com.android.multidex.MainDexListBuilder "${tmpOut}" ${@} ||  exit 11

</code>
</pre>


<p>
该脚本大体分为3步：</br>
1、环境检查，包括参数合法性检查，文件路经检查，proguard命令检查。</br>
2、调用proguard生成一个tmp jar包。</br>
3、将生成的tmp jar包和输入的文件组作为参数，调用com.android.multidex.MainDexListBuilder类生成文件列表。</br>
第一步的环境检查就不说了，下面主要分析下后面两步。
</p>

# 2.proguard生成入口类jar包 #
<p style="text-indent:2em;">
环境检查完成后，脚本调用了proguard命令来生成一个tmp jar包，
由于脚本中没有相关注释，刚开始看到这个步骤不知道是做什么的。
于是分析下命令的参数：</br></br>
injars：指定要处理的应用程序jar,war,ear和目录</br>
outjars：指定处理完后要输出的jar,war,ear和目录的名称</br>
libraryjars：指定要处理的应用程序jar,war,ear和目录所需要的程序库文件</br>
dontoptimize：不优化输入的类文件</br>
dontobfuscate：不混淆输入的类文件</br>
dontpreverify：不预校验</br>
dontwarn：缺省proguard 会检查每一个引用是否正确，但是第三方库里面往往有些不会用到的类，没有正确引用。如果不配置的话，系统就会报错。</br>
include：包含的规则文件，使用的proguad规则是sdk/build-tools/21.1.2/mainDexClasses.rules，内容如下：</br>
<pre>
<code>
 -keep public class * extends android.app.Instrumentation {
    <init>();
  }
  -keep public class * extends android.app.Application {
    <init>();
    void attachBaseContext(android.content.Context);
  }
  -keep public class * extends android.app.Activity {
    <init>();
  }
  -keep public class * extends android.app.Service {
    <init>();
  }
  -keep public class * extends android.content.ContentProvider {
   <init>();
  }
  -keep public class * extends android.content.BroadcastReceiver {
   <init>();
  }
  -keep public class * extends android.app.backup.BackupAgent {
   <init>();
  }

  -keep public class * extends java.lang.annotation.Annotation {
   *;
  }
</code>
</pre>
proguard官网绘制的执行流程图：</br>
<img src="https://raw.githubusercontent.com/coolpers/coolpers.github.io/master/assets/posts/2015-04-23-mainDexClasses/proguard-sequence.png" alt="" width="60%"/></br>
结合proguard的执行流程图及脚本中参数，发现只执行了shrink操作。</br></br>
下载proguard源码，通过查看proguard的代码，发现shrink步骤会根据keep规则保留需要的类，去除不需要的。</br></br>
至此，我们明白最后打出的tmp jar包只保留了符合keep规则的类，这些类作为入口类。</br></br>
验证：</br>
注释掉脚本中trap cleanTmp 0这一行，防止jar包在脚本执行完成后被删除。
运行脚本完成后在/tmp目录下多了一个mainDexClasses-xxxxx.tmp.jar文件，将文件用JD-GUI打开，如下：</br>
<img src="https://raw.githubusercontent.com/coolpers/coolpers.github.io/master/assets/posts/2015-04-23-mainDexClasses/proguard-sequence.png" alt="" width="60%"/></br>
只保留了符合keep规则的类。
</p>


# 3.MainDexListBuilder生成文件列表 #
<p style="text-indent:2em;">
虽然前面已经生成了入口类，但我们不能直接将入口类的类名输出到文件作为主dex的文件列表。比如入口类中的匿名内部类。</br>
这就需要再分析入口类，找出其引用的其他类，MainDexListBuilder实现了该功能，<a href="https://android.googlesource.com/platform/dalvik/+/893795fc95fdd77d398ebb77a0fe336c45b596cf/dx/src/com/android/multidex/MainDexListBuilder.java">下载MainDexListBuilder代码</a>。</br>
MainDexListBuilder构造方法：</br>
<pre>
<code>
 public MainDexListBuilder(String rootJar, String pathString) throws IOException {
        ZipFile jarOfRoots = null;
        Path path = null;
        try {
            try {
                jarOfRoots = new ZipFile(rootJar);
            } catch (IOException e) {
               ......
            }
            path = new Path(pathString);
            ClassReferenceListBuilder mainListBuilder = new ClassReferenceListBuilder(path);
            mainListBuilder.addRoots(jarOfRoots);
            for (String className : mainListBuilder.getClassNames()) {
                filesToKeep.add(className + CLASS_EXTENSION);
            }
            keepAnnotated(path);
        } finally {
          ......
        }
    }

</code>
</pre>
rootJar是入口类jar包，封装成一个ZipFile</br></br>
pathString是我们指定的输入文件组，封装成一个Path类，该类会遍历所有的的class文件，并根据class文件的格式解析，将class文件的各个部分映射到DirectClassFile类中,最后生成类名到DirectClassFile的哈希表。</br></br>
生成ClassReferenceListBuilder实例，调用addRoots方法，该方法会遍历入口类的jar包，获取入口类并将类名加入到classNames列表中，然后在Path中根据类名查询到该入口类对应的DirectClassFile，根据DirectClassFile常量表，查找该类引用到的其他类，并将引用到的类及其父类和接口加入到classNames列表中，具体如下：</br>
<pre>
<code>
    public void addRoots(ZipFile jarOfRoots) throws IOException {
        // keep roots
        for (Enumeration<? extends ZipEntry> entries = jarOfRoots.entries();
                entries.hasMoreElements();) {
            ZipEntry entry = entries.nextElement();
            String name = entry.getName();
            if (name.endsWith(CLASS_EXTENSION)) {
                classNames.add(name.substring(0, name.length() - CLASS_EXTENSION.length()));
            }
        }
        // keep direct references of roots (+ direct references hierarchy)
        for (Enumeration<? extends ZipEntry> entries = jarOfRoots.entries();
                entries.hasMoreElements();) {
            ZipEntry entry = entries.nextElement();
            String name = entry.getName();
            if (name.endsWith(CLASS_EXTENSION)) {
                DirectClassFile classFile;
                try {
                    classFile = path.getClass(name);
                } catch (FileNotFoundException e) {
                 ......
                }               addDependencies(classFile.getConstantPool());
            }
        }
    }
</code>
</pre>
最后将获取的classNames输出到文件。至此主dex所需的类列表生成了。
</p>
# 4. 遇到的问题 #
1. proguard源码分析，源码中使用了大量的装饰和访问设计模式，阅读起来比较难懂。
2. proguard injars中包含已经混淆过的jar包，处理时会报“Unknown verification type [*] in stack map frame”错误，解决方案参见http://sourceforge.net/p/proguard/bugs/420/。
3. class文件结构 http://1025250620.iteye.com/blog/1971213。

