---
layout: post
title: 用bash写一个迷你监控脚本
abstract: 用bash脚本监控进程
category: technical
permalink: simple-bash-monitor
author: 木逸辰
tags: [tech, bash]
---

### {{ page.title }}

最近使用百度云盘app时，经常占用cpu资源过高导致退出，白天还好，晚上睡觉了想让它自己下载些资源，结果一早起来发现运行一会儿就退出了。

于是决定写个监控脚本，检测app的进程状态，一旦发现程序退出了，就自动重新启动。先检测进程是否在运行：

	function detect(){
		#ps -ex 返回所有正在运行的进程信息，根据名称过滤
		#因为同时会返回后面那条 grep $1 指令的进程，所以再过滤一遍把这个去掉
		re=`ps -ex | grep $1 | grep -v grep`

		#判断结果是否为空，为空就代表已退出了，重新打开并记录日志
	    if [ -z "$re" ]; then
	        open $1
	        log
	    fi
	}

循环检测的代码（我为了省事儿直接把app名称写死这儿了，当然也可以使用变量，获得更好的灵活性）：

	function watch(){
		#每隔3秒检测一次
	    while true; do
	        detect "/Applications/BaiduNetdisk_mac.app"
	        sleep 3
	     done
	 }

最后是记录日志的代码：

	function log(){
		#记录写入 watcher.log文件
		if [ ! -f "watcher.log" ]; then
	        touch "watcher.log"
	    fi

		#获取当前时间并格式化
	    dt=`date '+%Y-%m-%d %H:%M:%S'`
	    echo "Restarted at $dt\n" >> watcher.log
	}

最后在文件结尾调用`watch`方法即可。这里注意的是，`bash`里的`=` 前后绝对不能有空格，不然会被当作命令，出现错误。另外`bash`里面对函数的调用必须在定义之后，所以整体代码如图：

![bash](/assets/images/2017-06-05-bash.png)

保存为`.sh`文件，运行即可

	$ ./watcher.sh &

最后的`&`符号是让进程在后台运行，不占用当前`terminal`窗口：

![bash](/assets/images/2017-06-05-bash-&.png)
