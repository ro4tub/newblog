---
layout: post
title: 安装私有Git服务
---
1. 为什么不用Github或者Gitlab

	Github收费略贵，Gitlab对于丐版linode跑起来有点吃力。

2. 安装git：

		sudo apt-get install git

3. 创建git用户

		sudo adduser git
4. 禁用git登录
		
		#修改/etc/passwd文件
		git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
		
5. 切换到git用户

		su - git
		
6. 把你开发机器的ssh public key上传到$HOME/oscar.pub
		
		#在**开发机**器执行
		cd ~/.ssh
		ssh-keygen -t rsa
		#把id_rsa.pub修改为oscar.pub上传到服务器

7. 安装gitolite
	
		git clone git://github.com/sitaramc/gitolite
		mkdir -p $HOME/bin
		gitolite/install -to $HOME/bin
		
8. 把自己设为管理者
		
		$HOME/bin/gitolite setup -pk oscar.pub

9. 增加新用户，通过gitolite-admin来管理的
		
		git clone git@host:gitolite-admin
		#在keydir目录增加alice.pub ssh公钥
		#编辑conf/gitolite.conf
			@gamedb = alice
			repo gamedb_data
				RW+ 	= @gamedb

		git add keydir
		git commit -m "added gamedb_data, gave access to alice"
		git push
	在clone代码的时候总是提示输入代码，参照[文档](http://gitolite.com/gitolite/sts.html#stsapp1)觉得没有问题，使用`ssh -vvv git@host info`发现问题出在/etc/ssh_config, 默认IdentityFile被设置为/User/oscar/.ssh/openssh-key
	
10. 通知alice下载代码

		git clone git@host:gamedb_data

11. 参考资料
	
	* https://github.com/sitaramc/gitolite
	* http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137583770360579bc4b458f044ce7afed3df579123eca000
	
