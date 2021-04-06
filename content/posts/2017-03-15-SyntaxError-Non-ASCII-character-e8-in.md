---
title: "Python-SyntaxError:Non-ASCII character '\\e8' in...."
date: 2017-03-15 22:53:51
tags:
  - Python
  - SyntaxError
categories: Python
---

## SyntaxError:Non-ASCII character '\e8' in File...
<!--more-->

遇到下面这种提示的错误

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/error.png)

修正方法：检查文件头部的UTF-8编码语句是否书写正确
```
	#coding=utf-8
```
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=27836503&auto=0&height=66"></ iframe>

今天下午同事梦鸽遇到一个问题，想直接把要找的一篇网页源代码中的特定的链接找出来，看到她需要每天手动复制粘贴出来然后一点一点删除，这样来找的话，效率也太低了。刚好我最近也刚开始学习python编程，何不写个脚本练练手？

所以就花了一个多小时帮梦鸽童鞋写了一个自动抓取链接的脚本，虽然开始一直头疼不会正则表达式该怎么写，但最终用笨办法：通过用

- 循环
- 条件
- 逻辑

这些基本的运算符，帮她实现了这个功能。

在这个过程中，遇到了最上面的这个错误，排查了半天以为是哪里的标点符号用成了中文字符，最后一想'Non-ASCII'意思就是找不到对应的ASCII码，既然找不到，是不是文件的编码格式错了，才想起来回头去看文件头部的

	#coding = utf-8

这句话，我居然写成了

	#condind=utf-8

改正确以后就没问题了。

很巧的是，同在一起学习python的QQ群同学，晚上也遇到了同样的问题，毫不犹豫的帮他找到了问题所在，顿时感觉半个多小时拍错过程是值得的。

特记录这篇日记，希望以后不要再出错。

###### 附python源码，抓取网页中CSDN博主的文章链接，并保存到blogID.txt文件中

```Python

	#coding=utf-8
	
	import re
	
	fp = open('sample.txt')#要打开的文件名
	temp = fp.read()
	blogID = open('BlogID.txt', 'w') #新建一个文件夹
	tmp_list = re.findall(r"(?<=href=\").+?(?=\")|(?<=href=\').+?(?=\')",temp)#检索出页面所有链接
	url_list = []
	for i in tmp_list:
	if ('blogdevteam' not in i) and ('cloumn' not in i) and ('article' in i) and ('http://blog.csdn.net' in i):
		if i not in url_list:
			url_list.append(i)
	#print url_list
	#blogID.write(str(url_list))
	for url in url_list:
	if url in url_list:
		print >> blogID , url
	blogID.close()
	fp.close()

```