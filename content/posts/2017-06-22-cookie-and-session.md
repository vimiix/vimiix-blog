---
title: 'Web|关于cookie和session一点知识'
date: 2017-06-22 16:03:54
tags:
  - Python
  - Django
  - cookie
  - session
categories: web
---

### cookie([维基百科定义](https://zh.wikipedia.org/wiki/Cookie))

> Cookie（复数形态 Cookies），中文名称为“小型文本文件”或“小甜饼”[1]，指某些网站为了辨别用户身份而储存在用户本地终端（Client Side）上的数据（通常经过加密）。

<!--more-->

### cookie 机制

cookie 机制采用的是在<font color="red">客户端保持状态</font>的方案。

cookie 机制，就是当服务器对访问它的用户生成了一个 session 的同时服务器通过在 HTTP 的响应头中加上一行特殊的指示以提示浏览器按照指示生成相应的 cookie，保存在客户端，里面记录着用户当前的信息，当用户再次访问服务器时，浏览器检查所有存储的 cookie，如果某个 cookie 所声明的作用范围大于等于将要请求的资源所在的位置也就是对应的 Cookie 文件,若存在，则把该 cookie 附在请求资源的 HTTP 请求头上发送给服务器。

cookie 的内容主要包括：名字，值，过期时间，路径和域。

前三个内容比较好理解，不作叙述。

路径就是跟在域名后面的 URL 路径，比如`/`或者`/foo`等等，路径与域合在一起就构成了 cookie 的作用范围。

###### cookie 生命周期

如果不设置过期时间，则表示这个 cookie 的生命期为浏览器会话期间，只要关闭浏览器窗口，cookie 就消失了。这种生命期为浏览器会话期的 cookie 被称为会话 cookie。会话 cookie 一般不存储在硬盘上而是保存在内存里，当然这种行为并不是规范规定的。

如果设置了过期时间，浏览器就会把 cookie 保存到硬盘上，关闭后再次打开浏览器，这些 cookie 仍然有效直到超过设定的过期时间。

<hr>
### session([百度百科定义](http://baike.baidu.com/link?url=gaQ1ddSuc9j2tPxoLnE8kQr7g2-KkMjqPe2pwAFPlQMLdWY6AqvjgclDupxfqkk2zeEkHkAgWCNhnZUSHXjMkq#1))

> Session 直接翻译成中文比较困难，一般都译成时域。在计算机专业术语中，Session 是指一个终端用户与交互系统进行通信的时间间隔，通常指从注册进入系统到注销退出系统之间所经过的时间。以及如果需要的话，可能还有一定的操作空间。

### session 机制

Session 机制采用的是在<font color="red">服务器端保持状态</font>的方案。

当用户访问到一个服务器，服务器就要为该用户创建一个 SESSION，在创建这个 SESSION 的时候，服务器首先检查这个用户发来的请求里是否包含了一个 SESSIONID，如果包含了一个 SESSIONID 则说明之前该用户已经登陆过并为此用户创建过 SESSION，那服务器就按照这个 SESSIONID 把这个 SESSION 在服务器的内存中查找出来（如果查找不到，就有可能为他新创建一个），如果客户端请求里不包含有 SESSIONID，则为该客户端创建一个 SESSION 并生成一个与此 SESSION 相关的 SESSIONID。这个 SESSIONID 是唯一的、不重复的、不容易找到规律的字符串，这个 SESSIONID 将被在本次响应中返回到客户端保存，而保存这个 SESSIONID 的正是 COOKIE，这样在交互过程中浏览器可以自动的按照规则把这个标识发送给服务器。

### Request 请求流程中的 cookie 和 session

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/cookie-session-1.png)

### Request 结束之后的流程

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/cookie-session-2.png)

### Django 中操作 cookie 和 session

```Python

	#设置cookie
	response.set_cookie("cookie_Key", "value", max_age)
	#获取cookie
	value = request.COOKIES["cookie_Key"]
	#删除cookie
	response.delete_cookie("cookie_Key", path="/", domain=name)

	#设置session
	request.session["session_name"] = "admin"
	#获取session
	session_name = request.session["session_name"]
	#删除session
	del request.session["session_name"]
```

<style>
table th:nth-of-type(1) {
    width: 100px;
}
table th:nth-of-type(2) {
    width: 100px;
}
</style>

##### 设置 cookie 可传递的可选参数描述：（参考自网络）

| 参数    | 缺省值 | 描述                                                                                                                                                                                                                                                       |
| :------ | :----- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| max_age | None   | cookies 的持续有效时间（以秒计），如果设置为 None cookies 在浏览器关闭的时候就失效了                                                                                                                                                                       |
| expires | None   | cookies 的过期时间，格式： "Wdy, DD-Mth-YY HH:MM:SS GMT" 如果设置这个参数，它将覆盖 max_age 参数。                                                                                                                                                         |
| path    | "/"    | cookie 生效的路径前缀，浏览器只会把 cookie 回传给带有该路径的页面，这样你可以避免将 cookie 传给站点中的其他的应用。当你的应用不处于站点顶层的时候，这个参数会非常有用。                                                                                    |
| domain  | None   | cookie 生效的站点。你可用这个参数来构造一个跨站 cookie。如,domain=".example.com"所构造的 cookie 对下面这些站点都是可读的： www.example.com 、www2.example.com 和 an.other.sub.domain.example.com 。如果该参数设置为 None ，cookie 只能由设置它的站点读取。 |
| secure  | False  | 如果设置为 True ，浏览器将通过 HTTPS 来回传 cookie。                                                                                                                                                                                                       |
