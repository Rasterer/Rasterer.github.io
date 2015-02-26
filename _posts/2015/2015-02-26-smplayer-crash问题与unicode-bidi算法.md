---
layout: post
title: "Smplayer crash问题与Unicode Bidi算法"
categories:
- 
tags:
- 


---
今天在Gentoo上装好smplayer之后，发现不能使用，一打开视频就报错： 

~~~
mplayer has finished unexpectedly exit code 1
~~~

但直接在命令行里敲命令却没有问题：

~~~
mplayer kauai_1080p_MPEG4_AVC_H.264_AAC_new.mp4
~~~

后来发现是smplayer调用mplayer中的一个选项有问题： `-noflip-hebrew` ，把这个选项去掉就OK了。接触过《圣经》的人肯定知道Hebrew是希伯来语的意思。
又看到在原始的log中有句：

~~~
MPlayer was compiled without FriBiDi support.
~~~

查了一下，知道FriBiDi是Unicode Bidirectional Algorithm (bidi)的开源实现。不同的语言有不同的书写方式，大部分从左往右书写，也有个别语言——如阿拉伯语和希伯来语，从右往左书写。某些时候，这两种书写方式可能混合存在。因此需要有一个”人“把文字的显示方向告诉计算机底层硬件。Bidi算法就是做这个事情的。

查到这里，smplayer的问题已经迎刃而解，只需要为mplayer添加一个USE.

~~~
media-video/mplayer bidi
~~~

不过由此引出的关于文字显示的问题还是很有趣的。这个一般在web前端领域会涉及的比较多。特别是我们中国的古籍都是从上往下，从右往左书写，大家有兴趣可以去了解一下如何将这种方式在计算机上显示出来。