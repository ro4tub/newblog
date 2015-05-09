---
layout: post
title: 移动终端(ios/android)下如何抓包
---

## iOS下抓包
原理是在MAC下使用内置工具rvictl创建虚拟网卡,和iOS设备互通，然后通过tcpdump或者wireshark在虚拟网卡上抓包。

具体步骤如下：

	$ifconfig -l #查看已有的网卡
	$rvictl -s 设备号 #安装虚拟网卡
	$ifconfig -l
	$sudo tcpdump -i rvi0 -n -s 0 -w dump.pcap #抓包
	$rvictl -x 设备号 #删除虚拟网卡

## Android下抓包

原理是在MAC/PC下使用adb安装和执行tcpdump工具，最后把抓下来的数据包文件下载到本地通过wireshark分析。和ios下不同的是，抓包行为直接发生在Android设备上。

MAC下具体步骤如下:
	
	$wget https://github.com/Trinea/trinea-download/raw/master/tcpdump #下载android版本的tcpdump
	$adb push tcpdump /sdcard/tcpdump
	$adb shell
	>su
	>mount -o remount,rw /system
	>cat /sdcard/tcpdump > /system/bin/tcpdump
	>chown root:shell /sytem/bin/tcpdump
	>chmod 754 /system/bin/tcpdump
	>mount -o remount,ro /sytem
	>rm /sdcard/tcpdump
	>tcpdump -n -s 0 -w /sdcard/dump.pcap
	$adb pull /sdcard/dump.pcap dump.pcap
	$open dump.pcap #使用wireshark打开分析

## 参考文档
1. [Technical Q&A QA1176](https://developer.apple.com/library/mac/qa/qa1176/_index.html)
2. [Installing tcpdump for Android](https://josetrochecoder.wordpress.com/2013/11/04/installing-tcpdump-for-android/)
