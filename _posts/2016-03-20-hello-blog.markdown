---
layout:     post
title:      "Hello blog!"
subtitle:   " \"Everything for a reason.\""
date:       2016-03-20 20:39:00
author:     "Don"
header-img: "img/black.jpg"
tags:
    - 生活
    - linux 
---


## 前言

[跳过废话，直接看技术实现 ](#build)

由于工作的关系最近在学习linux, 很难，很多东西看过之后不久就忘了，所以萌生了写博客的想法，以此帮助自己系统有组织的学习及归纳整理知识。 于是开始选址选方案，看了知乎上简书上以及百度搜索看到层出不穷各种花式方案，最后决定用 **github+jekyll+markdown** 这个看似有逼格有腔调适合程序猿折腾的个人主页式的博客。 
**但过程是曲折的**。

<p id = "build"></p>
---

## github

git是一个分布式版本管理系统，[github](https://github.com/)可以托管各种git库，很多开源项目都在上面托管，很好很先进。然而起初并不知道这具体是个什么东西，感谢简书作者AzureYu的博文[“授人以渔”的教你搭建个人独立博客](http://www.jianshu.com/p/8f843034c7ec)，写得真的很细致，让我这个小白此方案的技术实现有了个轮廓。也感谢来自廖雪峰的官方网站的[git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)，如作者所说**史上最浅显易懂的git教程**绝不夸张，让我对git的指令操作有了系统的学习熟悉。

简单整理几个步骤： 

* 注册github账号
* 建立一个仓库, 此仓库命名必须为**your-user-name.github.io**的方式，即为个人主页域名。
* 下载一个git shell终端工具，我用的是[git](https://git-scm.com/download/)，不同PC平台注意版本。 
* clone到本地进行修改，熟悉git的命令。

至此完成了github相关的工作。

---

## jekyll环境

> [Jekyll](http://jekyllrb.com/) is a simple, blog-aware, static site generator.
> [Jekyll](http://jekyllrb.com/) 是一个简单的博客形态的静态站点生产机器。

jekyll即是我们博客的内容来源，可生成静态网页上传后显示出来。搭建环境建议按照官方的[安装流程](https://jekyllrb.com/docs/installation/)来, 第一次搭建也是比较复杂的。

我的步骤是：

* [ruby](https://www.ruby-lang.org/en/downloads/)及devkit, 特别注意devkit的安装方式.
	{% highlight ruby linenos %}
	'cd devkit // 目的是将当前目录转移到devkit解压路径'
	'ruby dk.rb init'
	'在配置文件里面需要加入相应的路径'
	'ruby dk.rb install'
	{% endhighlight %}


* 通过gem install jekyll 命令安装jekyll的软件包，可能由于国外网站的关系下了，可以用[淘宝镜像源](https://ruby.taobao.org/)替换原有资源。
* 安装node.js, python软件。


至此所需环境已经搭建完毕，可以新建一个工程目录在本地调试。**jekyll工程文件的组织可参照官方网站**。

---

## 主题

如何选择一个称心如意的主题真是一件纠结的事情，由于本人不懂前端所以只能先借鉴别人的主题一用。 在此感谢[Hux黄玄](https://github.com/Huxpro/huxpro.github.io)的主题, 做得很时尚炫酷，一眼相中，果断先借来使用。 当然我也用了几天的时间在codecademy网站上学了学HTML5&CSS相关知识，以期后续能按照自己的需求加入new feature.

## markdown

用markdown写blog相信是一件非常惬意的事情，很简单的标记语言，参照[认识与入门markdown](http://sspai.com/25137), 就不多言赘述了。

## 结束
个人博客仓促上线，很多地方并未完善，但前后也花了近1个月的时间，学到了很多东西，但不忘初衷，尽早开始blog才是正途。

关于此blog规划：
* 博客主要关于技术，linux, c, c++, 以提升自己能力记录总结学习过程为主。
* 完善此博文，加入具体指令。
* 优化完善网页，按自己需求加入新特性，修改UI，头像等，流量分析等，前端涉及到的知识还是蛮多的。

## update

2016/3/21 22:31
V1.0 邮箱：randongchn@gmail.com