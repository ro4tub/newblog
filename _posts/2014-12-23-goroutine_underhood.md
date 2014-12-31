---
layout: post
title: Goroutine的内部实现
---


##一. 核心数据结构(G/P/M)

在最初的版本，只有G和M；直到1.1版本，为了解决性能问题，引入了P。

现在，G描述一个Goroutine的所有信息：栈、执行上下文、Groutine Id和状态信息；
P维护着运行中和运行完的Goroutine列表、内存池；
M对应一个OS线程，需要绑定P，才可以执行G.

另外，G/P/M分别通过schedlink、link和schedlink构成一个（侵入式）列表。

G/P/M定义在runtime.h文件，下面我列出其中的重要字段，并给出详细解释：
 
	struct  G
	{
	    uintptr stackguard0;    // 用于栈保护，但可以设置为StackPreempt，用于实现抢占式调度
	    uintptr stackbase;  // 栈顶
	    Gobuf   sched;		// 执行上下文，G的暂停执行和恢复执行，都依靠它
	    uintptr stackguard; // 跟stackguard0一样，但它不会被设置为StackPreempt
	    uintptr stack0;		// 栈底
	    uintptr stacksize;	// 栈的大小
	    int16   status;		// G的六个状态
	    int64   goid;		// G的标识id
	    int8*   waitreason; // 当status==Gwaiting有用，等待的原因，可能是调用time.Sleep之类
	    G*  schedlink;		// 指向链表的下一个G
	    uintptr gopc;       // 创建此goroutine的Go语句的程序计数器PC，通过PC可以获得具体的函数和代码行数
	};
	
	struct P
	{
	    Lock;		// plan9 C的扩展语法，相当于Lock lock;
	
	    int32   id;  // P的标识id
	    uint32  status;     // P的四个状态
	    P*  link;		// 指向链表的下一个P
	    M*  m;      // 它当前绑定的M，Pidle状态下，该值为nil
	    MCache* mcache;	// 内存池
	
	    // Grunnable状态的G队列
	    uint32  runqhead;
	    uint32  runqtail;
	    G*  runq[256];
	
	    // Gdead状态的G链表（通过G的schedlink）
		// gfreecnt是链表上节点的个数
	    G*  gfree;
	    int32   gfreecnt;
	};

	struct  M
	{
	    G*  g0;     // M默认执行G
	    void    (*mstartfn)(void);	// OS线程执行的函数指针
	    G*  curg;       // 当前运行的G
	    P*  p;      // 当前关联的P，要是当前不执行G，可以为nil
	    P*  nextp;	// 即将要关联的P
	    int32   id; // M的标识id
	    M*  alllink;    // 加到allm，使其不被垃圾回收(GC)
	    M*  schedlink;  // 指向链表的下一个M
	};

G执行结束后，并不会被删除，而放在gfree链表上，供以后创建重用。

P在启动的时候，按照GOMAXPROCS环境变量的值创建P，支持外部重设该值，但一般不会那么做。

M在启动的时候创建sysmon用于系统监控外，还会在需要的创建M，系统会确保一直有一个空闲的M，一般跟P的个数相当。创建好的M，不会被回收。

##二. G/P的状态迁移和G的执行
G/P的状态都是互斥的，不存在同时拥有两个状态。

G有六个状态: Gidle,Grunnable, Grunning, Gsyscall, Gwaiting, Gdead


下图是G的状态迁移图：

![image]({{ site.url }}/images/G_states.svg)

P有五个状态：Pidle, Prunning, Psyscall, Pgcstop, Pdead

下图是P的状态迁移图：

![image]({{ site.url }}/images/P_states.svg)

我们注意到，G的一条状态迁移的关键路径是:
Gidle(Gdead)->Grunnable->Grunning->Gdead
状态Gdead的G不会被释放，会被重复使用。


那么，当G从状态Grunning迁移到状态Gwaiting做了什么呢？
恢复执行的时候，从状态Gwaiting到状态Grunnable，再到状态Grunning又做了什么呢？

首先看一下Gobuf结构体的定义(只看前面三个字段)：

	struct  Gobuf
	{
	    // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
	    uintptr sp; // 栈指针
	    uintptr pc; // 程序计数器PC
	    G*  g; 	// 关联的G
	};

然后看两个重要的汇编函数：

	// 把执行上下文SP/PC保存到Gobuf中
	/ void gosave(Gobuf*)
	TEXT runtime·gosave(SB), NOSPLIT, $0-8
	      MOVQ    8(SP), AX       // gobuf
	      LEAQ    8(SP), BX       // caller's SP
	      MOVQ    BX, gobuf_sp(AX)
	      MOVQ    0(SP), BX       // caller's PC
	      MOVQ    BX, gobuf_pc(AX)
	      MOVQ    $0, gobuf_ret(AX)
	      MOVQ    $0, gobuf_ctxt(AX)
	      get_tls(CX)
	      MOVQ    g(CX), BX
	      MOVQ    BX, gobuf_g(AX)
	      RET
	
	// 从Gobuf中恢复状态，并跳转到PC，继续执行
	// void gogo(Gobuf*)
	TEXT runtime·gogo(SB), NOSPLIT, $0-8
	      MOVQ    8(SP), BX       // gobuf
	      MOVQ    gobuf_g(BX), DX
	      MOVQ    0(DX), CX       // make sure g != nil
	      get_tls(CX)
	      MOVQ    DX, g(CX)
	      MOVQ    gobuf_sp(BX), SP    // restore SP
	      MOVQ    gobuf_ret(BX), AX
	      MOVQ    gobuf_ctxt(BX), DX
	      MOVQ    $0, gobuf_sp(BX)    // clear to help garbage collector
	      MOVQ    $0, gobuf_ret(BX)
	      MOVQ    $0, gobuf_ctxt(BX)
	      MOVQ    gobuf_pc(BX), BX
	      JMP BX
	  

和swapcontext比起来，是不是觉得Goroutine的上下文切换的成本更低？
从状态Grunning到状态Gwaiting会执行gosave，保存上下文；等到状态Grunnable到状态Grunning的时候会执行gogo，恢复上下文并继续执行。


##三. Goroutine的调度
调度器的状态由Sched结构体维护，参与调度的是每一个M，M调用函数schedule后，永不退出，不断找寻可以运行的G(函数findrunnable), 找到后就开始运行（函数execute）。

我们先看一下，Sched的定义：

	struct Sched {
	    Lock;	// plan9 C扩展语法
	
	    uint64  goidgen; // 用于生成goroutine id
	
	    M*  midle;   // 空闲的M
	    int32   nmidle;  // 空闲M的个数
	    int32   nmidlelocked; // 被锁住的M的个数
	    int32   mcount;  // 当前总共M的个数
	    int32   maxmcount;  // 支持创建的最大个数，默认上限为10000
	
	    P*  pidle;  // Pidle状态的P链表头
	    uint32  npidle; // Pidle状态P的个数
	
	    // Grunnable状态的G的队列
	    G*  runqhead;
	    G*  runqtail;
	    int32   runqsize; // 队列长度
	
	    // Gdead状态的G的链表，当P的gfree链表大于64的时候，P会保留32个，把多余的都挂到这个链表上
		// 当P的gfree链表小于32的时候，P会尽量从这个链表上，移动部分过去，直到满32个
	    Lock    gflock;
	    G*  gfree;
	};


几个重要的全局变量：
	
	Sched   runtime·sched; // 调度器
	M*  runtime·allm;	// 所有M的链表头

	static  Lock allglock;  // 保护allg的锁
	G** runtime·allg;	// G的链表，新创建的G都会挂在上面
	uintptr runtime·allglen; // G链表的长度
	static  uintptr allgcap; // G链表的容量


三个核心函数：

	// 调度的一个回合：找到可以运行的G，执行
	// 从不返回
	static void
	schedule(void)
	{
	    G *gp;
	    uint32 tick;

	top:
	    gp = nil;
	    // 时不时检查全局的可运行队列，确保公平性
	    // 否则两个goroutine不断地互相重生，完全占用本地的可运行队列
	    tick = m->p->schedtick;
	    // 优化技巧，其实就是tick%61 == 0
	    if(tick - (((uint64)tick*0x4325c53fu)>>36)*61 == 0 && runtime·sched.runqsize > 0) {
	        runtime·lock(&runtime·sched);
	        gp = globrunqget(m->p, 1); // 从全局可运行队列获得可用的G
	        runtime·unlock(&runtime·sched);
	        if(gp)
	            resetspinning();
	    }
	    if(gp == nil) {
	        gp = runqget(m->p); // 如果全局队列里没找到，就在P的本地可运行队列里找
	        if(gp && m->spinning)
	            runtime·throw("schedule: spinning with local work");
	    }
	    if(gp == nil) {
	        gp = findrunnable();  // 阻塞住，直到找到可用的G
	        resetspinning();
	    }

		// 是否启用指定某M来执行该G
	    if(gp->lockedm) {
			// 把P给指定的m，然后阻塞等新的P
	        startlockedm(gp); 
	        goto top;
	    }

	    execute(gp); // 执行G
	}


	// 找一个可运行的G来执行
	// 尝试从其它P偷，也会从全局队列或者轮询网络中获得G
	static G*
	findrunnable(void)
	{
	    G *gp;
	    P *p;
	    int32 i;

	top:

	    if(runtime·fingwait && runtime·fingwake && (gp = runtime·wakefing()) != nil)
	        runtime·ready(gp);
	    // P的本地可运行列表中找
	    gp = runqget(m->p);
	    if(gp)
	        return gp;
	        
	    // 再去全局可运行列表中找
	    if(runtime·sched.runqsize) {
	        runtime·lock(&runtime·sched);
	        gp = globrunqget(m->p, 0);
	        runtime·unlock(&runtime·sched);
	        if(gp)
	            return gp;
	    }
	    // 轮询网络事件
	    gp = runtime·netpoll(false);  // 非阻塞
	    if(gp) {
	        injectglist(gp->schedlink); // 把gp链表挂到全局可运行链表上
	        gp->status = Grunnable;
	        return gp;
	    }
	
	    // 随机地从其它P偷G
	    for(i = 0; i < 2*runtime·gomaxprocs; i++) {
	        p = runtime·allp[runtime·fastrand1()%runtime·gomaxprocs];
	        if(p == m->p)
	            gp = runqget(p);
	        else
	            gp = runqsteal(m->p, p);
	        if(gp)
	            return gp;
	    }
	stop:
	    // return P and block
	    runtime·lock(&runtime·sched);
	    if(runtime·sched.runqsize) {
	        gp = globrunqget(m->p, 0);
	        runtime·unlock(&runtime·sched);
	        return gp;
	    }
	    p = releasep(); // 返回可用的P
	    pidleput(p);
	    runtime·unlock(&runtime·sched);
	    if(m->spinning) {
	        m->spinning = false;
	        runtime·xadd(&runtime·sched.nmspinning, -1);
	    }
	    // 再一次检查所有P的状态
	    for(i = 0; i < runtime·gomaxprocs; i++) {
	        p = runtime·allp[i];
	        if(p && p->runqhead != p->runqtail) {
	            runtime·lock(&runtime·sched);
	            p = pidleget();
	            runtime·unlock(&runtime·sched);
	            if(p) {
	                acquirep(p);
	                goto top;
	            }
	            break;
	        }
	    }
	    // 轮询网络
	    if(runtime·xchg64(&runtime·sched.lastpoll, 0) != 0) {
	        if(m->p)
	            runtime·throw("findrunnable: netpoll with p");
	        if(m->spinning)
	            runtime·throw("findrunnable: netpoll with spinning");
	        gp = runtime·netpoll(true);  // 阻塞，直到有网络事件	        runtime·atomicstore64(&runtime·sched.lastpoll, runtime·nanotime());
	        if(gp) {
	            runtime·lock(&runtime·sched);
	            p = pidleget();
	            runtime·unlock(&runtime·sched);
	            if(p) {
	                acquirep(p);
	                injectglist(gp->schedlink);
	                gp->status = Grunnable;
	                return gp;
	            }
	            injectglist(gp);
	        }
	    }
	    stopm();
	    goto top;
	}



	// 使G在当前的M上运行
	// 从不返回
	static void
	execute(G *gp)
	{
	    if(gp->status != Grunnable) {
	        runtime·printf("execute: bad g status %d\n", gp->status);
	        runtime·throw("execute: bad g status");
	    }
	    gp->status = Grunning;
	    gp->waitsince = 0;
	    gp->preempt = false;
	    gp->stackguard0 = gp->stackguard;
	    m->p->schedtick++;
	    m->curg = gp;
	    gp->m = m;
		// 恢复gp里保存的上下文，继续执行
		// gp执行完毕后，会调用goexit0，继续调用schedule
		// 这就是所谓的从不返回
	    runtime·gogo(&gp->sched);
	}

看完近两百行代码后，我们来思考几个问题？

1. 要是某个G长时间占用CPU，怎么办?
2. Goroutine之间会出现资源竞争的，那么锁问题怎么解决？死锁怎么避免？

问题1，go支持简单的抢占式调度，当sysmon线程检测到长时间运行的G，会把它抢占过来，具体代码见函数retake

问题2，可以通过Sync.Mutex/Sync.RWMutex加锁，同时go工具链提供竞争检测功能。另外，runtime会调用函数checkdead检测死锁，一旦存在，就会结束程序。


##四. 栈的设计

我们先看一下栈的大体结构：

	   +------------------+
	   | args from caller |
	   +------------------+ <- frame->argp
	   |  return address  |
	   +------------------+ <- frame->varp
	   |     locals       |
	   +------------------+
	   |  args to callee  |
	   +------------------+ <- frame->sp
  

问题来了，我们给goroutine的栈分配多大比较合理呢？

4K?

大于4K的goroutine怎么办？

go1.2开始支持栈的动态调整，采用split stack方法，一开始分配比较小的栈，等某调用需要扩大栈的时候，分配另一个栈，并跟当前的栈组成一个链表，当调用结束后，归还扩大的栈。

下面这个程序就会不断分配栈，释放栈，怎么办？

	func main() {
	    for {
	        big()
	    }
	}

	func big() {
	    var x [8180]byte
	    // do something with x

	    return
	}

go1.3改为采用contiguous stack的方法，当发现某个调用导致栈不够的时候，就会分配一个大一点的栈，并把原来栈上的数据memmove过去，最后重新调整指针，使其指向新的地址。

go连接器默认会在函数序言（function prolog）中增加侦测栈是否足够的代码，一旦不够，就会调用morestack来扩大栈。NOSPLIT修饰的函数除外。


##五. 总结

上面对于Goroutine的内部实现做了详细的剖析，如果您想直观地看一下它们的状态，不妨自己动手，编译下面的程序，并通过GOMAXPROCS=2 GODEBUG="schedtrace=1000,scheddetail=1" ./goroutines启动。

	// goroutines.go
	package main
	import (
	    "fmt"
	    "sync"
	    "time"
	)

	func main() {
	    wg := sync.WaitGroup{}
	    for i:=0; i<1000000; i++ {
	        wg.Add(1)
	        go func(wg sync.WaitGroup, i int) {
	            defer wg.Done()
	            time.Sleep(100*time.Millisecond)
	        }(wg, i)
	    }

	    wg.Done()
	    fmt.Println("finished")
	}


你会看到如下的日志：

	SCHED 0ms: gomaxprocs=2 idleprocs=0 threads=4 idlethreads=0 runqueue=1805 gcwaiting=0 nmidlelocked=1 nmspinning=0 stopwait=0 sysmonwait=0
	  P0: status=1 schedtick=1 syscalltick=0 m=0 runqsize=186 gfreecnt=0
	  P1: status=0 schedtick=2 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
	  M3: p=-1 curg=-1 mallocing=0 throwing=0 gcing=0 locks=0 dying=0 helpgc=0 spinning=0 blocked=0 lockedg=-1
	  M2: p=-1 curg=17 mallocing=0 throwing=0 gcing=0 locks=0 dying=0 helpgc=0 spinning=0 blocked=0 lockedg=-1
	  M1: p=-1 curg=-1 mallocing=0 throwing=0 gcing=0 locks=1 dying=0 helpgc=0 spinning=0 blocked=0 lockedg=-1
	  M0: p=0 curg=16 mallocing=0 throwing=0 gcing=0 locks=1 dying=0 helpgc=0 spinning=0 blocked=0 lockedg=-1
	  G16: status=2(garbage collection) m=0 lockedm=-1
	  G17: status=3() m=2 lockedm=-1
	  G18: status=4(GC sweep wait) m=-1 lockedm=-1
	  G19: status=1() m=-1 lockedm=-1
	  G20: status=1() m=-1 lockedm=-1
	  G21: status=1() m=-1 lockedm=-1
	  G22: status=1() m=-1 lockedm=-1
	  
第一行数据是调度器整体信息：

	0ms：时间戳
	gomaxprocs：  跟GOMAXPROCS环境变量一致，P的个数
	idleprocs：空闲P的个数
	threads：OS线程个数，也是M的个数
	idlethreads：空闲的M个数
	runqueue：G运行队列的长度
	
Pn行都是有关P的信息：
	
	status：P的状态
	schedtick：被调度的次数
	syscalltick：调用系统调用的次数
	m：关联M的id
	runqsize：G运行队列长度
	gfreecnt：Gdead状态的G链表长度
	
Mn行都是有关M的信息：
	
	p：关联P的id
	curg：当前运行的G
	

Gn行都是有关G的信息：

	status：G的状态
	m：关联M的id
	lockedm：锁住的M
	
这些信息对于你查找性能问题或者属性go调度器都会很有帮助。

##六. 参考资料

1. [Contiguous stacks](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub) 
2. [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw)
3. [NUMA-aware scheduler for Go](https://docs.google.com/document/d/1d3iI2QWURgDIsSR6G2275vMeQ_X7w-qxM2Vp7iGwwuM/pub)
4. [race detector](http://blog.golang.org/race-detector)
5. 深入理解计算机系统（第二版）