---
layout: post
title: Go1.3 GC日志格式
---

通过环境变量GODEBUG=gctrace=1启动程序，会在stdout打印GC日志。

$ GODEBUG=gctrace=1 go run gc.go


	gc1(1): 4+0+183+0 us, 0 -> 0 MB, 18 (19-1) objects, 0/0/0 sweeps, 0(0) handoff, 0(0) steal, 0/0/0 yields
	gc2(1): 0+0+130+0 us, 0 -> 0 MB, 368 (369-1) objects, 12/0/0 sweeps, 0(0) handoff, 0(0) steal, 0/0/0 yields
	gc3(1): 0+0+213+0 us, 0 -> 0 MB, 437 (505-68) objects, 35/0/0 sweeps, 0(0) handoff, 0(0) steal, 0/0/0 yields
	gc4(1): 0+0+11941+0 us, 0 -> 30 MB, 529 (659-130) objects, 43/0/0 sweeps, 0(0) handoff, 0(0) steal, 0/0/0 yields
	scvg0: inuse: 2, idle: 0, sys: 3, released: 0, consumed: 3 (MB)
	scvg0: inuse: 31, idle: 0, sys: 32, released: 0, consumed: 32 (MB)
	scvg1: inuse: 2, idle: 0, sys: 3, released: 0, consumed: 3 (MB)
	gc5(1): 5+1+470+0 us, 30 -> 30 MB, 2709 (2943-234) objects, 45/2/0 sweeps, 0(0) handoff, 0(0) steal, 0/0/0 yields
	scvg1: GC forced
	scvg1: inuse: 31, idle: 0, sys: 32, released: 0, consumed: 32 (MB)



gc日志说明：

| 数据列 | 描述 | 
| ------------ | ------------- |
| gcA(B) | 从程序启动开始第A次gc, B个工作线程参与GC  |
| A -> B MB | A: 上次GC后堆栈占用大小 B: 这次GC前堆占用大小（包含垃圾）  |
| A+B+C+D us | ABCD分别表示stop-the-world, 清除(sweeping), 标记(marking)和等待工作线程完成时间，单位微秒  | 
| A (B-C) objects | ABC分别表示堆上总的对象个数（包括垃圾对象），总的内存分配和释放操作数  |
| A/B/C sweeps | ABC分别表示memory span总个数，被主动需要清除或者背景清除的memory span个数，stop-the-world阶段清除的memory span个数  |
| A(B) handoff | 并行标记阶段，A个handoff操作，B个对象被handoff  |
| A(B) steal | 并行标记阶段，A个steal操作，B个对象被stolen  |
| A/B/C yields | 并行标记阶段：等待其它线程期间有A+B个yield操作  |

scvg是runtime把内存还给操作系统，默认每2分钟执行一次。

| 数据列 | 描述 | 
| ------------ | ------------- |
| sys |  从操作系统申请的内存，跟top里的VSS一致 |
| inuse |  整个堆占用的内存（包括死的对象） |
| idle |  GC后就会变成未使用，长时间处于idle，就会归还给操作系统 |
| released | 这次归还的内存大小  |
| consumed |  当前实际使用的内存大小 |