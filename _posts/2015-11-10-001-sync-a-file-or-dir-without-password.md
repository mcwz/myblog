---
layout: post
title: rsync通过ssh免密码同步单个文件或者目录
description: rsync在配置同步时，通常是以同步文件目录的形式存在，同步时做验证时也是通过rsync自有的用户验证居多，本文介绍一种同步单个文件的方式，同时可以达到免除密码同步
keywords: rsync,ssh,文件,同步,目录
---

## 背景信息

最近要热备一个运行了接近10年的图片库程序，这个WEB程序在10年间已经积累了接近4T的文件，文件数量达百万个以上，想达到热备，决定采用inotify+rsync来完成文件的热备。在热备的过程中发现，启动rsync后，计算文件差异严重影响了现有程序的运行，因此后来决定，rsync只负责同步正在变化的文件，之前的文件统一用其他方式拷贝。

## rsync遇到的问题

rsync 常见的几种同步方式有以下几种

* rsync [OPTION]... SRC DEST 
* rsync [OPTION]... SRC [USER@]HOST:DEST 
* rsync [OPTION]... [USER@]HOST:SRC DEST 
* rsync [OPTION]... [USER@]HOST::SRC DEST 
* rsync [OPTION]... SRC [USER@]HOST::DEST 
* rsync [OPTION]... rsync://[USER@]HOST[:PORT]/SRC [DEST]

说实话，之前一直没理解单个冒号和两个冒号的区别，归根结底是没懂，远程shell的意思，于是一直把重点都放到了两个冒号的，结果发现两个冒号的没法制定目的服务器的路径，只填写一个目的服务器配置的module名称，具体路径在module里已经定义好了。这样在传单个文件时就出现了一个矛盾的事：如果不指定具体目的地的路径，那么rsync会把所有的文件都传到配置文件指定的一个路径下，原有的路径就丢失了，如果加了-R参数，那么rsync会重新建一个路径，这个路径的规则是你以rsync配置文件里的路径加上原机器的路径为备份路径，这显然也不对，然后备份进入了死胡同。

## 解决方案

偶然间看见了，rsync [OPTION]... SRC [USER@]HOST:DEST 中的单个冒号的意思是通过ssh的方式鉴权，然后恍然懂了：单个冒号是用ssh方式鉴权，两个冒号的意思是通过rsync鉴权的(自己猜的，不一定很准确)。而通过ssh鉴权的话，目标路径是可以任意指定的，通过rsync方式鉴权方式貌似只能按着目标机器的配置文件里的module路径去同步。

因此果断采用ssh方式。具体步骤如下：

### 先打通ssh免密码登录

先执行ssh-keygen，一路回车即可。

然后把生成的key拷贝的目标机器：ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.100.100

然后再执行rsync -avz -e ssh /home/testuser/somefile testuser@192.168.100.100:/backup/testuser/somefile

### 配合inotify 发送变化的文件

<pre class="prettyPrint">
/usr/local/inotify/bin/inotifywait -mr --timefmt '%d/%m/%y %H:%M' --format '%T %w %f' -e close_write,modify,delete,create,attrib $src |  while read DATE TIME DIR FILE; do
       FILECHANGE=${DIR}${FILE}

       /usr/bin/rsync -aqRH --delete  --progress ${FILECHANGE} root@192.168.100.100:$FILECHANGE
done
</pre>