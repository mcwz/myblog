---
layout: post
title: 开启shiro remember me功能
description: 使用shiro作为权限管理的WEB程序可以开启remember me功能，这样就可以使得登录过的用户无需再次登录即可。
keywords: shiro ,remember me
---

##开启方式

登录成功的时候可以增加以下设置

<pre class="prettyPrint">
Subject currentUser = SecurityUtils.getSubject();
UsernamePasswordToken token = new UsernamePasswordToken(username, password);
if(remeberMe==1)
{//记住我
	token.setRememberMe(true);
}
try {
	currentUser.login(token);
} catch (Exception e) {
}
</pre>

我在增加了以上设置后不管怎么设置记住我都没用。后来猜测是cookie的问题，于是又研究shiro的cookie设置。

##shiro的cookie设置

在shiro.ini里增加一行

<pre class="prettyPrint">
securityManager.rememberMeManager.cookie.name = remeberme
</pre>

终于，这次在FireBug里看到这个key值为remeberme的cookie。

此时，系统相当于已经记录了登录信息，下面就是在访问需要登录的页面的时候读一下这个cookie就行了。当然读cookie这事也得交给shiro来做。

##真正完成自动登录

其实，在登录的时候记录一个cookie只完成了这个功能的30%。重点还在后边。需要给所有受登录保护的地址增加一个全局的拦截器。在拦截器里读取这个cookie，验证没问题的话就允许登录即可。代码大致如下。

<pre class="prettyPrint">
Subject currentUser = SecurityUtils.getSubject();  
if (currentUser.isAuthenticated()) {  
	//已经登录的
	//允许正常操作
}
else
{//没有登录的，看看有自动登录的cookie没
	if(currentUser.isRemembered())
	{
		//有自动登录的cookie,那么调取shiro的登录过程再登录一次。
		String principal=currentUser.getPrincipal().toString();
		Users user=um.getUserByUsername(principal);
		UsernamePasswordToken token = new UsernamePasswordToken(user.getUsername(), user.getPassword(),true);
		currentUser.login(token);
	}
}
</pre>