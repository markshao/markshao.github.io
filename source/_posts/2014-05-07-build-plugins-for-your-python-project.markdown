---
layout: post
title: "build plugins for your python project"
date: 2014-05-07 21:06:12 +0800
comments: true
categories: python
---

第一次产生开发支持插件这个需求是因为[Pagrant](https://github.com/markshao/pagrant)这个项目,我希望可以让Pagrant通过插件的方式来支持各种不同的IaaS平台。不过之前的Python的开发的经验更多地是集中在Web开发领域，所以也没有插件开发方面的经验。查了一些Google的资料，也实现了一个非常粗糙的插件的支持。

提到Python的插件，不得不先提一下Python的模块的两个打包的工具`distribute`和`setuptools`,目前用的比较多的是`setuptools`这个工具，它需要在你的项目的根目录里面有一个`setup.py`的文件，然后他可以根据我们在setup.py中编写的内容来打包成module.

在setup.py的文件中，有一个关键字叫`entry_point`,这个就是我们用来实现插件的关键,这里放一个LabManager插件的setup.py的内容来看一下如何来实现一个插件的描述


``` python

setup(
    name="labmanager",
    version=__version__,
    packages=find_packages(),
    description='the labmanager cloud plugin for pagrant',
    classifiers=[
        "Programming Language :: Python",
    ],
    platforms='any',
    keywords='framework testing',
    entry_points={
        'PAGRANT': [
            "VMPROVIDER=labmanager.actions:LabManagerVmProvider",
            "VMPROVIDER_INFO=labmanager:get_provider_info"
        ]
    }

```

这个setup.py文件中指定了两个接口的定义，一个是`VMPROVIDER`还有一个是`VMPROVIDER_INFO`,这个分别对应的是项目代码里面的一个函数和一个实现类.

当我们安装这个LabManager的模块的时候，就会自动把这个ENTRY_POINT注册到了Python的解释器的安装目录里面,我们可以通过`pkg_resource`提供的API来实现对ENTRY_POINT的查找和调用

``` python
vmprovider_class = load_entry_point(self.vmprovider_info.get("type"), "PAGRANT", "VMPROVIDER")
```

不过在Pagrant项目里面有一个问题，我怎么查找和管理我们现在安装的插件呢，我目前采用的方式是在当前项目的目录下写一个隐藏文件`.plugin`,这个带来了一个问题就是当我们更改了目录以后，就没法查找到插件了。

还需要寻找一个更好的插件管理的方案
