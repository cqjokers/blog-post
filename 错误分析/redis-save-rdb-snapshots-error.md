---
title: >-
  Redis ＂MISCONF Redis is configured to save RDB snapshots, but is currently not
  able to persist on disk＂问题
tags:
  - redis
  - error
categories:
  - 错误分析
abbrlink: 9ef81304
date: 2018-08-14 13:36:22
---
今天中午，朋友说之前我做的那个外包项目客户那边说数据不对。于是马上登录服务器把日志下载下来查看原因，查看日志发现出现了如下图所示的问题：![](https://i.loli.net/2018/08/14/5b72656d33de9.png)日志里出现了大量的"MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error."错误，于是再去查看redis日志，发现出现了"Can’t save in background: fork: Cannot allocate memory"错误提示。对于第一条错误，网上很多都是“config set stop-writes-on-bgsave-error no”但这样做仅仅是忽略了这个异常，使得程序能够继续往下运行，但实际上还是会失败！再查看redis日志的时候还发现了一条这样的提示：![](https://i.loli.net/2018/08/14/5b726772228fb.png)（警告：过量使用内存设置为0！在低内存环境下，后台保存可能失败。为了修正这个问题，请在/etc/sysctl.conf 添加一项 'vm.overcommit_memory = 1' ，然后重启（或者运行命令'sysctl vm.overcommit_memory=1' ）使其生效。）这样的提示在我的印象中好像基本上每次重启redis的时候都有出现，但当时都是直接忽略掉并没有放在心上。结合这条提示大概就能明白为什么会出现"Can’t save in background: fork: Cannot allocate memory"的错误，人家明明都已经提示了而我却没有引起注意只能怪自己太SB，最后按照提示进行修改，在`/etc/sysctl.conf`中加入`vm.overcommit_memory=1`保存后并执行命令`sysctl vm.overcommit_memory=1`使其生效，重启redis后即可，至于`vm.overcommit_memory=1`这是干嘛用的就只有去问娘了。
> 内核参数overcommit_memory 
> 它是 内存分配策略
>可选值：0、1、2。
0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
2， 表示内核允许分配超过所有物理内存和交换空间总和的内存