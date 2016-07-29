---
layout: post
title: 关于Apache MIME和MIME_Magic模块
description: 今天要给Discuz程序里做一个模块，这个模块可以直接在帖子里显示PDF文档，部署过程中发现运行环境不能正确显示，于是研究了一番，发现还是与MIME有关。
keywords: Apache , MIME , MIME_Magic
categories: [记录]
---

## 事件经过

今天要给Discuz程序里做一个模块，这个模块可以直接在帖子里显示PDF文档，部署过程中发现运行环境不能正确显示。表现为在Chrome浏览器下一直显示加载文件中。查看console中也没有什么报错。因为以前遇到过其他文件类似的情况，于是猜测是MiME的问题，于是去查Apache配置，奇怪的是Apache里指定了PDF相关的内容，然后线索就断了。

期间查看了Chrome 开发者工具中的network面板中发现Chrome多发了状态为206的请求，觉得可能是这的原因，但是查完后206的涵义后还是毫无头绪。后来认真看了一下Respone Header里的内容发现 content type竟然是text/html 这就奇怪了，明明是pdf啊。于是焦点就又回到了MIME上边。因为在测试环境是没问题的，于是把测试环境和运行环境的httpd.conf里关于mime的内容搜索出来详细对比了一下，发现测试环境中关于MIME_Magic的模块是加载的，运行环境是注释了，莫非问题出在这里？于是将测试环境关于MIME_Magic的配置拷贝过去，reload apache后问题解决。

解决这事几乎耗了一下午。详细搜索了一下MIME_Magic，下面详细说一下mod_mime和mod_mime_magic。


## Apache模块 mod_mime

浏览器在接收到回应的时候是需要知道具体是哪种content type才能正确解析的。为此Apache提供了mod_mime模块来处理部分常见的文件类型。

Apache手册上时这样描述的：

> 本模块通过文件的扩展名将不同的"元信息"与文件关联起来。元信息在文档的文件名与文档的MIME类型、语言、字符集、编码方式之间建立关联。

以上说明mod_mime模块通过文件后缀名来检测文件的类型，同时把处理结果发给浏览器。问题就出在这里了，Discuz上传后的PDF不再是以PDF为后缀，而是以.attach为后缀，导致Apache识别不出文件类型只能以默认的text/html输出。解决此问题mod_mime_magic就上场了。

## Apache模块 mod_mime_magic

手册上说：

> 本模块采取Unix系统下file(1)命令相同的方法：检查文件开始的几个字节，来判定文件的MIME类型。

因此对于后缀名不是PDF的attach，Apache通过读取文件仍然能够识别此文件的正常类型。

## 还需要验证的

1. mod_mime和mod_mime_magic的验证顺序，比如说我把.txt的文件后缀名改为PDF后不知道能不能正确识别。
2. mod_mime_magic在执行完毕后得到是PDF的结论后还需不需要在调取mod_mime模块来获取具体的输出类型，因为具体的输出类型在mime.types记录的。