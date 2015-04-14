---
layout: post
title: 关于jQuery与JavaScript的事件绑定
description: 在jQuery里摸爬滚打了一阵子后，感觉对JQ也算是比较来电了，谁知，今天遇到一件事，感觉还是有点力不从心，因此特意在此处记下来。
keywords: jQuery , JavaScript , 事件绑定
---

以下属于当时的牢骚，想看正文的可以忽略，goto label:正文。 

在jQuery里摸爬滚打了一阵子后，感觉对JQ也算是比较来电了，谁知，今天遇到一件事，感觉还是有点力不从心，因此特意在此处记下来。 

事情是这样滴：我在做一个鼠标移上去执行某个动作的操作。当然，这肯定是用mouseover事件，结果该执行的代码没有执行，本着懒人的态度，我没有去详细分析原因，猜测应该是 DOM是后加载过来的，那么干脆直接用bind算了，bind一个应该没有问题吧，结果还不行，于是我就晕了。无可奈何之下，只能百度（不要拍我，我也想用google,不过实在是太慢了，有时候搜到了还打不开），关键词是这样的“jQuery 绑定事件失效”。结果我搜到了一般关于live方法的文章。文章太长了，又有代码，不好理顺了，继续懒人，直接翻一下手册，看看live。结果发现正好合适，live可以想像成bind的bind的（当然实际上不是），于是用了一下，果然可以了。高兴之余，记录之。


* 正文: *

## Javascipt的事件绑定 

这个可以使用一些DOM自带的绑定方式：如 

`<a href="#" onclick="runthis()"></a>`


如果您的JS代码里有对应的runthis方法，那么这个就可以执行了。 


这么用的好处是一目了然，看一下就知道点击的话会执行哪个方法，而且问题少，只要DOM在，这个事件就会被绑定；缺点是JS代码和HTML混到一起了。说白了就是耦合度太高，哪天不需要这个click事件了，要改html。 


当然Javascipt也可以直接在自己的代码里做绑定，但原理就和jQuery的一样了，那么干脆直接说jQuery的事件绑定。 

## jQuery事件绑定

基础绑定obj.click()

这个绑定是理解起来最简单的事件绑定 


手册里是这么写的 


`$("p").click( function () { $(this).hide(); }); `

这样的话，点击P后会隐藏掉自己。 


这种情况下，如果DOM一直没有增加的话，完全能应付。然而这个DOM是如果在页面初始化的时候不存在后来又加进来的咋办呢？ 


这时候就到了bind上场了。 


## 稍微灵活点的事件绑定obj.bind('click',function(){})

这个稍微灵活一些，你可以随时执行这个方法，前提是只要这个DOM在执行这段话的时候已经存在的，那么你就可以处理了，一定要记得哈，这个DOM存在之后才能绑。过了这个村就跟bind没关系了。 


`$("p").bind("click", function(){ 
  alert( $(this).text() ); 
}); `

然而我这人又懒了，我不想在执行绑定的事件的时候看看这个DOM有了没，我想开始的时候就加上它，什么时候有了这个DOM都可以执行就可以了。 


这时候NB的live出厂了。 


## NB的绑定事件的方法live

`obj.live('click',function(){}); `


这个看起来是和bind长的很像的，但是是bind的变种。书上说，这家伙在不管当前DOM有没有都行。也就是说后期你加入的DOM照样能绑上！ 


`$('.clickme').live('click', function() { 
  alert("Bound handler called."); 
}); `

就像这样，原来的时候你有一些class叫做clickme的可以运行，执行完这句后有新加的clickme也行！ 