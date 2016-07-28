---
layout: post
title: 使用单例模式解决mmseg4j在高并发时导致内存溢出问题
description: 使用 Lucene 做了一个站内搜索，分词器是 mmseg4j 。此搜索不仅用作正常搜索之用，同时借助搜索完成了相关文章的功能。因相关文章处在内容页，并发量很高，本文主要记录在高并发的情况下我是如何解决了 mmseg4j 的内存问题。
keywords: mmseg4j,Lucene,高并发,内存,单例
categories: [开发, 入门]
---

##之前的项目

公司之前的站内搜索是通过爬虫爬取后，然后通过 Lucene 分词完成的搜索。此种方式有个缺点就是，明明数据可以直接读到，反而用爬虫去爬取，自己人给自己人增加压力，同时因为页面千差万别，导致数据采取并不是很快，同时有些文章也采不到。另外这个搜索是当时买别的系统赠的，爬虫时不时还会罢工很长时间。基于此，部门决定自己写搜索。

##新搜索的构架

打算采用 Lucene + mmseg4j 来完成核心的索引以及分词。考虑到相关文章并发量比较大，增加ehcache作为缓存，key采用搜索关键词，这样的话，只要搜索词不变的话，就无需重新搜索，可以直接读取内存结果了。

##部署以及遇到的问题

第一版处理了约15G的数据，搜索单个词的速度还算可以，同时测试了一个并发量比较小的频道的相关文章，发现还算胜任，然后就又找了一个并发量相对较大的频道部署，成功后发现mmseg4j每次搜索的时候都在 try to load ... 详细看了一下是在读字典文件，而这个在并发量小的时候还好，当并发量大起来之后，满屏都在 try to load ，过一会 tomcat 就开始报错了，提示java heap space ,同时还伴有IO错误，猜测应该是并发上来之后，IO性能可能不够了。考虑了一下，词典这东西应该是不需要每次都加载啊，只要我的词典内容不变，没必要非重新加载一遍这个，于是就找mmseg4j的配置，发现只有new Analyzer的时候和mmseg4j有关。于是就考虑单例了，想法就是Tomcat初始化时我new一下mmseg4j，之后就不在new了，如果 mmseg4j 只是在每次构造的时候加载词典的话，那么应该就没问题了，于是就着手把mmseg4j分词器改成单例模式。

##单例模式之路

首先做如下单例定义：

<pre class="prettyPrint">
public class SingleAnalyzer {
	private static class SingletonHolder 
	{
		static String dicFilePath=PathKit.getWebRootPath()+File.separator+"_dicdata"+File.separator;;
		private static final Analyzer INSTANCE = new ComplexAnalyzer(dicFilePath);  
	}  

	private SingleAnalyzer (){}  
	
	public static final Analyzer getInstance() {  
	    return SingletonHolder.INSTANCE;  
	}  
}
</pre>

然后在tomcat生命周期的对象里增加如下定义：

<pre class="prettyPrint">
public static Analyzer singleAnalyzer=SingleAnalyzer.getInstance();
</pre>

然后在使用时就不要new Analyzer了，直接通过长生命周期的类去调用就行了。

##写在最后

目前对单例还不是很熟，现在上线测试了几个小时了，还没发现问题。