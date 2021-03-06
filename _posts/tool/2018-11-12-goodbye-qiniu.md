---
layout: post
title: 拜拜了，七牛。GitHub 才是图床的王道
category: 工具
tags: GitHub  图床
---
* content
{:toc}

刚开始搭建博客的时候，就在想着图床从哪里找，最开始是使用有道云笔记当图床，后来觉得麻烦，然后网上很多人都建议使用七牛，什么10G免费存储之类的，觉得还挺好，就用七牛了，可是谁知道，最近收到七牛的邮件，说什么我的测试域名回收通知，如果正式域名，还要什么公安网备案，太麻烦。我就自己写个博客（尽管我好长时间不更新一次），偶尔贴张图片，又不想那么商业化，咋找个免费的这么难啊。难道真的满足不了我这小小需求吗？忽然想到，GitHub好像就行。Google吧。然后就看到了使用GitHub当做图床教程，方法很简单。

在自己的 blog 文件夹目录下面新建一个专门用来存放图片的文件夹，例如就叫 images
如下图
![](../../../../images/github_image_1.png)
放入的图片怎么引用呢？

查看 `_site` 这个文件夹下面，就会出现一个 images 文件夹。
如下图
![](../../../../images/github_image_2.png)
而写 blog 一般都是以一个日期开头的，例如我这篇blog 文件名 是 `2018-11-12-goodbye-qiniu.md`。 那么生成网页的文件路径就是：  `2018/11/12/goodbye-qiniu/index.html`
![](../../../../images/github_image_3.png)
index.html 后退四次，即  ‘../../../../’ 就和 images 文件夹 在同一个目录了。

所以添加一张图片，只需要
<font color="#ff000" >
![](../../../../images/图片名)<br>
![](../../../../images/图片名)<br>
![](../../../../images/图片名)<br><br>
</font>
重要的事情说三遍。 这样就行了
例如上面三张，包括下面这张，都是使用这中格式添加图片。
![](../../../../images/github_image_4.png)


这样在本地预览的时候，查看图片地址 `http://localhost:4000/images/github_image.png`，加载的是本地图片。

真正提交到 github 上，查看图片地址就变成了  `http://hoyouly.fun/images/github_image.png`，加载网络路径。

这样以后写blog的时候，想贴图片，就简单了，先弄张图片，扔到 images 文件里面。
那么路径就是`../../../../images/图片名`， 路径有了，那就插入图片吧。 最后一块提交，齐活。  

GitHub 才是王道，如果连GitHub都不能用了，那还敲个写个啥博客啊，直接回家种地不就行了。

---
搬运地址：    
