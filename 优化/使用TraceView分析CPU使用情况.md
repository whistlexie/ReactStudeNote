## 使用TraceView分析CPU使用情况 ##

在一进入 React 页面的瞬间，CPU会突然飙高，对比原生要多上20%左右（具体数据，不同机型表现不一样），实际上，在性能测试项目中，Native 和 React 的页面，UI复杂程度基本一致的，甚至可以认为是使用的相同的UI（React多了几层，例如ReactRootView这一层），但是预料中的性能代价不至于这么大，初步判断是JS和android通信导致的性能代价，下面使用 TraceView 进行分析。

### TraceView 介绍 ###


### 使用方式介绍 ###

在代码中使用Debug类埋点，进行数据分析。在需要开始分析的地方加入  Debug.startMethodTracing("calc")，代码，这里传递的参数是生成的文件名，可传可不传，然后在结束的地方加入代码     Debug.stopMethodTracing()。当你start的时候，便会创建一个prof文件，开始记录对应线程的方法执行情况以及执行时间，stop的时候，分析结束，然后你可以使用 adb命令 adb pull /sdcard/文件名.trace /tmp 获prof文件，然后再用DDMS打开。
直接打开DDMS，然后点击按钮 start method profiling，然后可以选择两种模式，一种是间隔采样模式，一种是自由模式，接着你需要停止分析，就按下 stop method profiling 按钮，就可以看到分析结果界面了。

某次操作结果如下显示：

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/%E5%AE%8C%E6%95%B4.png)

### 分析结果界面介绍 ###

这里将界面分为上下两部分，上部分是 TimelinePanel，上部分的左边条目栏是这次分析的线程信息，每一行对应一个线程，包含了线程名称的线程PID，然后右边是 time line，记录了分析时间内，每个线程上的方法执行情况和执行时间。提供位置选择，你可以通过鼠标去确定特定线程的特定时刻的方法执行情况，然后在下部分会显示你定位的那个方法相关信息，在刻度线上边也会显示方法的一些信息。同一时刻，一个线程只有一个执行的方法。

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/timeplante.png)

然后下部分是，各种方法的具体执行情况，下面是一些概念介绍

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/ProfiePanel.png)

- Parent 调用当前方法的方法
- Children 被当前方法调用的方法

表格参数如下图：

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/table.png)



**一些注意事项：**

- android 4.4 以后提供一种新的分析方式 sample-base profiling，间隔采样分析。
- 如果使用 debug 类，在2.1以前需要在 androidmainfest文件中增加文件读取权限。





如何分析：

初步的打算是，这样的是根据 prof 文件和 APT 分析，用APT找出CPU飙高的那个时刻，然后回到 prof 文件中查找对应的时刻，查看对应时刻执行了什么方法。单纯的根据 prof文件，可能还查看不了 CPU 过高的问题。这里要考虑的是，APT和DDMS不能同时使用，所以需要留意时间一致性。下面是 APT的CPU使用情况，多次尝试发现，点击进入 React 页面，CPU就会飙高，然后大概65秒左右CPU就回到低耗状态了。

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/APT.png)

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/APT02.png)




然后，这里去到 TraceView 去分析代码（这里最好需要关闭程序，重新进入保证测试的准确性），测量结果如下显示，但是这里显示发现，在 main 线程里面调用的方法，只有


然后我们将 excul cpu time 排一下序，分别测试 Native 和 React 页面

Native 页面：

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/Native.png)


React 页面：

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/React.png)


这里改变一下思路，从CPU被消耗的角度去查找一些方法，主要分析下面这两种方法：

- 调用次数多的方法，切换方法会造成一些性能消耗。
- 总执行时间长的方法，执行时间长，直接占用了CPU资源。

这两种的指标图示如下：

执行次数排行

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/React%20%E6%89%A7%E8%A1%8C%E6%AC%A1%E6%95%B0%E5%A4%9A.png)

执行时间排行

![](https://github.com/whistlexie/ReactStudeNote/blob/master/%E4%BC%98%E5%8C%96/TraceView/React%E6%89%A7%E8%A1%8C%E6%97%B6%E9%97%B4%E9%95%BF.png)




其实可以单纯的参考一个指标 Excl CPU time%，查看的

对比发现，React 消耗CPU的主要方法如下：

1. Handler的handleCallBack()方法，执行了76次，总共370毫秒，实际占用CPU时间，12.1%
1. DirectByteBuffer的 checIsAccessible()方法和checkNotFreed()方法，分别执行了8400和8800次，总共87毫秒和60毫秒。
1. 然后是nio里面的一堆方法，看来要着重分析这一块了
1. ArtMethod.getMethodName()方法，执行1631次，总共43毫秒，实际占用CPU 1.4%。
1. Dex$Section.readByte（）方法，执行5122次，总共34毫秒
1. FinznilzerDaemon.doFinalize（）
1. String和Object 一些方法
1. ViewManagerPropertyUpdater.updateProps（）方法，执行206次，总共23毫秒
1. TextView的构造方法


然而 Native 页面消耗CPU的主要方法却是：

BitmapFactory.decodeStream()方法，执行3次，总共39毫秒。
ArtMethod.getMethodName()方法。执行1487次，总共39毫秒。
资源获取，style 获取，执行907次，总共32毫秒
TextView 构造方法


简单的总结一下：

DirectByteBuffer 类是Java nio框架里面，一个用于字节缓存的类，在文件读取和网络操作中，这里的缓存是写在 C heap上的，所以速度非常之快。
Handler 的 handleCallBack()方法，






参考资料


- [Profiling with Traceview and dmtracedump](https://developer.android.com/studio/profile/traceview.html#traceviewLayout) TraceView 介绍
- [Traceview Walkthrough](https://developer.android.com/studio/profile/traceview-walkthru.html) TraceView 使用简单教程