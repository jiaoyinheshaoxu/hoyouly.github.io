---
layout: post
title: Android 内存管理机制
category: 读书笔记
tags: Android 内存管理
---
* content
{:toc}

## 内存分为以下几种

### 前台进程
目前屏幕上正在显示的进程和一些系统进程。举个例子：Dialer Storage，Google Search 等系统进程就是前台进程；再举例来说，当你运行一个程序，如浏览器，当浏览器界面在前
### 可见进程
就是不在前台，但是用户依然可见的进程，例如 widget，输入法，都属于可见进程，部分进程虽然不在前台，但与我们的使用也密切相关，我们也不希望它们被终
### 次要进程
目前正在运行的一些服务，（主要服务，如拨号等，是不可能被进程管理终止的），举例来说：谷歌企业套件，Gmail 内部存储，联系人内部存储等。这部分服务虽然属于次要服务，但很一些系统功能依然息息相关，我们时常需要用到它们，所以也太希望他们被终止。
### 后台进程
就是通常我们所说的启动后被切换到后台的进程，当程序显示在屏幕上时，他所运行的进程即为前台进程（foreground），一旦我们按 home 返回主界面（注意是按 home，不是按 back），程序就驻留在后台，成为后台进程（background）。
后台进程的管理策略有多种：
* 有较为积极的方式，一旦程序到达后台立即终止，这种方式会提高程序的运行速度，但无法加速程序的再次启动；
* 也有较消极的方式， 尽可能多的保留后台程序，虽然可能会影响到单个程序的运行速度，但在再次启动已启动的程序时，速度会有所提升。
这里就需要用户根据自己的使用习惯找到一个平衡点。

### 内容供应节点
没有程序实体，进提供内容供别的程序去用的，比如日历供应节点，邮件供应节点等。在终止进程时，这类程序应该有较高的优先权。
### 空进程
没有任何东西在内运行的进程，有些程序，比如 BTE，在程序退出后，依然会在进程中驻留一个空进程，这个进程里没有任何数据在运行，作用往往是提高该程序下次的启动速度或者记录程序的一些历史信息。这部分进程无疑是应该最先终止的

## 内存管理机制
Android 是一个多任务系统。当退出程序时，Android 系统不会立即将它 Kill，这样下次再运行该程序时，可以快速启动。随着系统中保留的程序越来越多，内存会出现不足，此时 Android 系统使用”LowMemory Killer”实现内存管理机制，即根据程序的重要性来决定 Kill 谁。Android 将程序的重要性分成以下几类，按照重要性依次降低的顺序：

![](http://img.blog.csdn.net/20150801184651864)

其中**每个程序都会有一个 oom_adj 值，这个值越小，程序越重要，被 Kill 的可能性越低**。除了上述程序重 要性分类之外，Android 系同还维护着另外一张表，这张表是一个对应关系，以 N1为例：

![](http://img.blog.csdn.net/20150801184846139)

这个表是定义了一个对应关系，每一个警戒值对应了一个重要性值，当系统的可用内存低于某个警戒值时，就杀掉所有大于该警戒值对应的重要性值的程序。
比如说，
当可用内存小于 6144 * 4K = 24MB 时， 开始杀所有的 EMPTY_APP，
当可用内存小于 5632 * 4K = 22MB 时，开始 Kill 所有的 CONTENT_PROVIDER 和 EMPTY_APP。上面这张对应表是由两个文件组成的：
/sys/module/lowmemorykiller/parameters/adj 和/sys/module/lowmemorykiller/parameters/minfree。
使用“ alter minfreee”可以修改 /sys/module/lowmemorykiller/parameters/minfree 这个文件。例如，若将最后一项改为 32 * 1024，那么当可用内存小于 128MB 是，就开始杀所有的 EMPTY_APP。



摘抄：

[Android内存管理机制](http://blog.csdn.net/hexieshangwang/article/details/47188987)
