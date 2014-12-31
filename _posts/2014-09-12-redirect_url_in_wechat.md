---
layout: post
title: 如何让App分享到微信里的链接直接转到App Store
---

很久以前，不需要做特殊处理，默认就支持。但是最新的版本，微信对于站内的链接做了很多控制。于是有人给出[iOS 7 新版微信 URL 不支持跳转 App Store 的解决方案](http://dearb.me/archive/2013-11-07/ios7-weixin-unsupport-redirect-to-app-store/)。但体验不好，还需要额外的开发量。

这里我推荐一个完美的方案，测试可以使用。在链接前添加前缀`http://mp.weixin.qq.com/mp/redirect?url=`

比如说，我的应用[【天天吞噬你】](http://itunes.apple.com/cn/app/tian-tian-tun-shi-ni/id902463169?l=zh&ls=1&mt=8)在App Store的链接为: 

	http://itunes.apple.com/cn/app/tian-tian-tun-shi-ni/id902463169?l=zh&ls=1&mt=8 

那么在代码中设置的url参数前面添加:
	
	http://mp.weixin.qq.com/mp/redirect?url=

最终变为: 

	http://mp.weixin.qq.com/mp/redirect?url=http://itunes.apple.com/cn/app/tian-tian-tun-shi-ni/id902463169?l=zh&ls=1&mt=8
	
	
Richard Stallman曾说：

>Programming is not a science. Programming is a craft