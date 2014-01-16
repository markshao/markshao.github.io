---
layout: post
title: "我们为什么开发Pagrant"
date: 2014-01-15 21:26:38 +0800
comments: true
categories: pagrant
---

估计大家都还不知到什么是Pagrant吧，哈哈。那就先老王卖瓜啦，Pagrant是一个完全基于虚拟化的分布式测试平台，它的目标是让我们的测试开发人员只需要把精力专注于分布式测试的本身，而不用去关心到底使用那个平台来支撑整个分布式测试的环境. Pagrant会自动的把测试运行在基于Linux Container , KVM，OpenStack, vCloud 等虚拟化的平台上。

Pagrant想法的出现源于那次参加了Pivotal团队的分享。在听完了Tony的演讲以后，我开始理解他们所描述的分布式测试概念的重要价值。其实我之前在参与到xDB的项目的时候，就已经开始困惑于如何在一个真正的分布式的平台上去实现Multi-Node的测试。虽然我后来做了一个[Tangram](http://www.infoq.com/cn/presentations/automated-test-environment-deploy-based-on-virtualization-technology),但是感觉这个只是专注于环境的部署，而不是一个针对测试而生的东西。

当我决定写一个Pagrant的平台的时候，我很纠结于如何让我们的测试可以轻松的迁移。比如我们在LXC的平台上调试和编写我们的测试代码，但是最后的回归测试是跑在OpenStack平台的。无缝迁移是这个系统所必须要支持的。就在这个时候感谢Samuel的精彩演讲，让我认识到了[Vagrant](http://www.vagrantup.com)的强大。

{% img /images/vagrant_logo.png %}

[Vagrant](http://www.vagrantup.com)，从读音就可以看出来，Pagrant和它有着密不可分的关系。Pagrant借鉴了非常多的Vagrant的设计，比如

* 使用Python文件而不是xml作为配置文件，充分利用脚本语言的动态能力
* entry-point的实现支持第三方vmprovider的添加和删除

当然Pagrant的自身也有它的特点

* 支持本地文件的插件方式，轻松修改你的虚拟化实现方式
* 使用Python开源测试框架[Nose](https://nose.readthedocs.org/en/latest/)作为引擎，支持Nose 所有特性
* 原生支持Linux Container，让测试虚拟化变得更加轻量化

目前Pagrant的最新版本已经可以基本支持以上的一些特性了，在这里要特别感谢晓波的大力支持。我们正在完善vmprovider的第三方插件体系的支持。如果您有兴趣我们的项目，也非常欢迎来我们的[项目页面](https://github.com/markshao/pagrant)提交您的一些建议和代码.

广告贴一枚
