---
layout: post
title: 一群理想主义开发者的天生骄傲 —— Go
---

老罗说，他是理想主义者，还说他以后再也不孤军奋战，会带上他的小伙伴一起。

老罗说，他天生骄傲。

我这里说的那群理想主义开发者，当然不是老罗，是Rob Pike、Ken Thompson、Robert Griesemer、Ian Lance Taylor和Brad Fitzpatrick等Go语言的核心开发者。

## Go的诞生
2007年9月，在听完一个讲解C++0x特性的Google内部分享后，rob拉上robert，还有ken，一边等待手头C++程序编译完，一边开始吐槽C++。然后不由分说地在白板上涂鸦出一门理想的开发语言，列出一堆想要的特性，这就是Go编程语言的起点。

rob整理了一个清单，相较于C/C++的不同之处:

1. 规范的语法（不需要解析符号表）
2. 垃圾回收
3. 没有头文件
4. 显式依赖
5. 没有循环依赖
6. 常量只能是数字
7. int和int32是不同类型
8. 字母大小写指定可见性
9. 任何类型都有方法（没有类）
10. 没有子类型继承（没有子类）
11. 等等

这就是开发者的天生骄傲，你觉得它不行，那就自己创造出一个行的。

## Go的工具链以及工具链的编译过程
一开始接触Go的开发者，会觉得所有的工具链都集成在go命令行工具里，`go build a.go` 编译a.go文件，生成a可执行文件。
随着深入学习，知道6g/6a，go命令行工具只是做了一个封装。

代表不同的硬件的数字：

数字 | 硬件平台 | 
------------ | ------------- | 
5 | ARM  | 
6 | AMD64  | 
8 | 386  |

不同的工具：

工具  | 输入 | 输出| 说明
------------ | ------------- | ------------| ------------
g | .go  | .5/.6/.8| 把go源代码生成go目标文件
a | .s  | .a| 把plan9汇编语言的代码编译成.a文件
l | .5/.6/.8  | 可执行文件|把go目标文件打包成目标平台可执行文件，例如linux平台的elf格式

Linux AMD64一般使用工具: `6g/6a/6l`

可以使用-x参数查看具体的执行过程，例如：`go build -x a.go`

那么工具链是怎么实现的呢？

rob等人都有写操作系统和C编译器经验的，他们天生骄傲。不依赖于Makefile等已有的编译系统，直接在make.bash里完成了编译环境的检测和所有工具链的编译。

1. 编译工具cmd/dist, 就这么任性,直接调用gcc：
		
		gcc -m64 -O2 -Wall -Werror -o cmd/dist/dist -Icmd/dist -DGOROOT_FINAL=\"/home/oscar/installer/go/\" cmd/dist/*.c
		
2. 编译go_bootstrap:

		
		cmd/dist/dist bootstrap -a -v
	

	生成了:
	
		oscar@ubuntu:~/installer/go/src$ cmd/dist/dist bootstrap -a -v
		lib9
		libbio
		liblink
		misc/pprof
		cmd/cc
		cmd/gc
		cmd/6l
		cmd/6a
		cmd/6c
		cmd/6g
		pkg/runtime
		pkg/errors
		pkg/sync/atomic
		pkg/sync
		pkg/io
		pkg/unicode
		pkg/unicode/utf8
		pkg/unicode/utf16
		pkg/bytes
		pkg/math
		pkg/strings
		pkg/strconv
		pkg/bufio
		pkg/sort
		pkg/container/heap
		pkg/encoding/base64
		pkg/syscall
		pkg/time
		pkg/os
		pkg/reflect
		pkg/fmt
		pkg/encoding
		pkg/encoding/json
		pkg/flag
		pkg/path/filepath
		pkg/path
		pkg/io/ioutil
		pkg/log
		pkg/regexp/syntax
		pkg/regexp
		pkg/go/token
		pkg/go/scanner
		pkg/go/ast
		pkg/go/parser
		pkg/os/exec
		pkg/os/signal
		pkg/net/url
		pkg/text/template/parse
		pkg/text/template
		pkg/go/doc
		pkg/go/build
		cmd/go





3.  安装标准库：

		../pkg/tool/linux_amd64/go_bootstrap install -v std

如果你感兴趣可以看一下每个工具是怎么实现的，目前大部分都是用C语言实现，1.4/1.5版本会用go改写。

go1.3版本的工具代码和入口如下：

1.	cmd/dist的入口在unix.c
2.	6g的代码在src/cmd/gc和src/cmd/6g, 入口在lex.c
3.	6l的代码在src/cmd/ld和src/cmd/6l，入口在lex.c
4.	6a的代码在src/cmd/6a，入口在lex.c
4.	go命令行工具的代码在src/cmd/go，用go语言实现的，入口在main.go

## Go的跨平台实现和跨平台交叉编译
"Write Once, Run Anywhere"是当年Sun宣传Java语言的一句口号。和Java不一样，Go并不是基于虚拟机实现跨平台的；甚至，Go还可以跨平台交叉编译，比如说在Linux操作系统里可以编译出在Windows下运行的PE文件。

只有天生骄傲的人，才会这么做。定义了一套go的汇编指令集和寄存器，解决平台的差异性。

在编译阶段，工具会把go文件、c文件或者s文件通过writeobj(定义在liblink/objfile.c)生成go目标文件。

然后，在链接阶段,把关联go目标文件通过asmb(定义在cmd/6l/asm.c)生成目标平台的可执行文件.
对于Linux的elf格式，会调用asmbelf(定义在cmd/ld/elf.c)生成可执行文件；对于Windows的pe格式，会调用asmbpe(定义在cmd/ld/pe.c)生成；对于Mac的macho格式，会调用asmbmacho(定义在cmd/ld/macho.c)生成。

我们来看一段go汇编语言实现的加法:

	// func Add(a int64, b int64) int64 // 跟Go代码混编的函数声明
	TEXT ·Add(SB), $0  // Add前面的不是点，而是unicode U+00B7
	MOVQ 	a+0(FP), BX // 类似linux汇编语言，左边是源，右边是目标，把第一个参数放到BX寄存器
	MOVQ	b+8(FP), BP // 把第二个参数放到BP寄存器, MOVQ操作64bit的整数
	ADDQ	BP, BX // 把BP和BX加起来，放到BX
	MOVQ 	BX, ·noname+16(FP) // noname前面的不是点，而是unicode U+00B7, 返回值紧跟在参数后面，两参数占16字节，所以16(FP)就是返回值地址
	RET

把上面代码放在add_amd64.s中，再创建一个文件add.go,代码如下:
	
	package math
	func Add(a int64, b int64) int64
	
然后就可以像调用其它go方法那样调用math.Add。
汇编代码由工具6a处理，语法文件定义在cmd/6a/a.y。

之前提到跨平台编译，需要执行如下步骤:

1. 重新编译Go源代码，如果想在Linux下编译Windows可执行文件，那么使用参数
	
		CGO_ENABLED=0 GOOS=linux GOARCH=amd64 ./make.bash
2. 使用如下方法编译自己的Go代码

		CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build a.go



## Go程序不依赖libc
我们也经常说尽量少依赖外部的库，我们的出发点是：一切尽在控制。但不管怎样都绕不开libc。
rob他们天生骄傲，拒绝依赖外部库，包括libc。
于是你会发现Go程序的内存分配直接使用mmap，线程创建直接使用clone，mutex锁直接使用futex。

这里我们详细分析一下线程创建过程。

Go使用者是无法创建线程的，Go标准库没有提供相关接口（直接调用系统调用除外）。什么时候创建线程，创建多少个线程都是由goroutine调度器控制的。不管怎样，都是通过runtime.newosproc完成的。
代码如下:

	void
	runtime·newosproc(M *mp, void *stk)
	{
    	int32 ret;
    	int32 flags;
    	Sigset oset;

	    /*
	     * note: strace gets confused if we use CLONE_PTRACE here.
	     */
	    flags = CLONE_VM    /* share memory */
	        | CLONE_FS  /* share cwd, etc */
	        | CLONE_FILES   /* share fd table */
	        | CLONE_SIGHAND /* share sig handler table */
	        | CLONE_THREAD  /* revisit - okay for now */
	        ;
	
	    mp->tls[0] = mp->id;    // so 386 asm can find it
	    if(0){
	        runtime·printf("newosproc stk=%p m=%p g=%p clone=%p id=%d/%d ostk=%p\n",
	            stk, mp, mp->g0, runtime·clone, mp->id, (int32)mp->tls[0], &mp);
	    }
	
	    // Disable signals during clone, so that the new thread starts
	    // with signals disabled.  It will enable them in minit.
	    runtime·rtsigprocmask(SIG_SETMASK, &sigset_all, &oset, sizeof oset);
	    ret = runtime·clone(flags, stk, mp, mp->g0, runtime·mstart);
	    runtime·rtsigprocmask(SIG_SETMASK, &oset, nil, sizeof oset);
	
	    if(ret < 0) {
	        runtime·printf("runtime: failed to create new OS thread (have %d already; errno=%d)\n", runtime·mcount(), -ret);
	        runtime·throw("runtime.newosproc");
	    }
	}
	
runtime.clone定义在sys_linux_amd64.s中，用汇编实现：

	// int64 clone(int32 flags, void *stack, M *mp, G *gp, void (*fn)(void)); // 函数声明
	TEXT runtime·clone(SB),NOSPLIT,$0
    MOVL    flags+8(SP), DI
    MOVQ    stack+16(SP), SI

    // Copy mp, gp, fn off parent stack for use by child.
    // Careful: Linux system call clobbers CX and R11.
    MOVQ    mm+24(SP), R8
    MOVQ    gg+32(SP), R9
    MOVQ    fn+40(SP), R12
    
    MOVL    $56, AX 	// clone系统调用
    SYSCALL

    // In parent, return.
	// 2(PC)意思是下面第二条指令
	// 如果返回值为0，那么运行下面的MOVQ SI, SP; 否则RET
    CMPQ    AX, $0
    JEQ 2(PC)	
    RET

    // In child, on new stack.
    MOVQ    SI, SP

    // Initialize m->procid to Linux tid
    MOVL    $186, AX    // gettid系统调用
    SYSCALL
    MOVQ    AX, m_procid(R8)

    // Set FS to point at m->tls.
    LEAQ    m_tls(R8), DI
    CALL    runtime·settls(SB)

    // In child, set up new stack
    get_tls(CX)
    MOVQ    R8, m(CX)
    MOVQ    R9, g(CX)
    CALL    runtime·stackcheck(SB)

    // Call fn
    CALL    R12 	

    // It shouldn't return.  If it does, exit
    MOVL    $111, DI
    MOVL    $60, AX		// exit系统调用
    SYSCALL
    JMP -3(PC)  // keep exiting

## 总结

rob他们是一群可爱的理想主义开发者，其实我们离他们并不远。我们可以在[github](https://github.com/golang/go)上看到他们每一次的递交，在[codeview](https://go-review.googlesource.com)里看到他们毫不留情地拒绝他人的递交，附上详细的理由。希望有一天，我也会成为那个天生骄傲的理想主义开发者。



## 参考资料

1. http://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html
2. https://golang.org/doc/asm
3. http://plan9.bell-labs.com/sys/doc/asm.html





    
    
    