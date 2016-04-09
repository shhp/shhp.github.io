---
layout: post
title: 抓取Android底层crash及其背后的秘密
---

做Android开发时，有时出于对程序运行效率或者跨平台的考虑，会使用NDK创建`so`共享库。由于共享库是用`C`以及`jni`进行开发，所以发生在底层共享库中的crash会比发生在`Java`层的crash更难抓取。这篇文章将介绍一下如何抓取共享库中的crash以及背后涉及的一些编译和链接的知识点。

<!-- more -->

### 如何抓取底层crash

在Linux系统中**信号(signal)**是进程间通信的一种基本方式。信号实质上是一种软中断。当一个用户进程发生了致命的crash的时候，系统会向该进程发送一个信号。因此我们可以通过捕获信号的方式来抓取Android的底层crash.

Linux提供了一个系统调用`sigaction`让用户进程可以捕获信号。`sigaction`的函数签名如下：

```
int sigaction(int signum, const struct sigaction *act,
              struct sigaction *oldact);
```

Linux man page对`sigaction`的解释是这样的：

> The sigaction() system call is used to change the action taken by a process on receipt of a specific signal. 

> signum specifies the signal and can be any valid signal except SIGKILL and SIGSTOP.

> If act is non-NULL, the new action for signal signum is installed from act. If oldact is non-NULL, the previous action is saved in oldact.

第一个参数`signum`是具体信号对应的整型值。比如常见的`SIGSEGV`对应的值是11. 第二个和第三个参数都是一个`sigaction`结构体的引用。`sigaction`的定义如下：

	struct sigaction {
	    void     (*sa_handler)(int);
	    void     (*sa_sigaction)(int, siginfo_t *, void *);
	    sigset_t   sa_mask;
	    int        sa_flags;
	    void     (*sa_restorer)(void);
	};

其中有两个函数指针`sa_handler`和`sa_sigaction`，它们都是在进程接收到信号后对信号进行处理的回调函数。根据`sa_flags`的设置选择其中一个进行信号处理。这里摘取一下man page的解释：

> If SA_SIGINFO is specified in sa_flags, then sa_sigaction (instead of sa_handler) specifies the signal-handling function for signum. This function receives the signal number as its first argument, a pointer to a siginfo_t as its second argument and a pointer to a ucontext_t (cast to void *) as its third argument.

这里我们选择`sa_sigaction`，因为它包含更多的参数，通过这些参数提供的信息，我们可以还原出发生crash时的函数调用栈。

至于还原函数调用栈，主要是参考了Android中的[libcorkscrew](https://android.googlesource.com/platform/system/core/+/jb-mr2-release/libcorkscrew/)库的源代码，这里不做详述。

###C程序的编译和链接

####ELF文件

当我们用`gcc`去编译一个`C`工程的时候，通常每一个`.c`文件都会对应生成一个`.o`目标文件。这些`.o`目标文件是一种被称作ELF(Executable and Linking Format)格式的文件。

一般而言，ELF文件可以分成三类：

1. 可重定位的对象文件(Relocatable file)
一般是由汇编器汇编生成的`.o`文件。编译之后，链接器(link editor)会以若干`.o`文件作为输入，经链接处理后，生成一个可执行的对象文件 (Executable file) 或者一个可被共享的对象文件(Shared object file)。

2. 可执行的对象文件(Executable file)
像`ls`、`vim`这样的二进制可执行文件。

3. 可被共享的对象文件(Shared object file)
这些就是所谓的动态库文件，例如`.so`文件。

一个可重定位的对象文件会包含若干个**section**. 一些常见的section有：

- .text section 里装载了可执行代码；

- .data section 里面装载了被初始化的数据，例如初始化过的全局变量；

- .bss section 里面装载了未被初始化的数据，因为没有数据，实际上这个section在文件中并不占空间。当可执行程序被执行的时候，动态链接器会在内存中开辟一定大小的空间来存放这些未初始化的数据，里面的内存单元都被初始化成0.

链接器在链接可执行文件或动态库的过程中，它会把来自不同可重定位对象文件中具有相同属性(比方都是只读并可加载的)的section合并成所谓**segment**.

###符号表

在一个可重定位`.o`文件中有一个比较重要的section叫符号表。它记录了源文件中声明的变量名以及函数名等符号的一些信息。下面用一个例子来说明。

首先创建一个`test.c`文件，包含源代码如下：

	int a = 2;
	int b;

	int main(int argc, char *argv[])
	{
	  return 0;
	}

执行`gcc -o test.o test.c`生成一个目标文件`test.o`. 在Mac系统中可以用`nm`这个命令来打印目标文件的符号表信息。运行`nm test.o`得到如下输出

	0000000100000000 T __mh_execute_header
	0000000100001000 D _a
	0000000100001004 S _b
	0000000100000f80 T _main

输出有三个部分：

1. 第一个部分是一串数字表示该符号对应的值，可以理解为是该符号对应的虚拟内存地址。
2. 第二个部分是一个字母，表示符号的类型。比如`T`表示该符号位于代码段，一般是一个函数名；`D`表示该符号位于初始化数据段。这些字母还有大小写之分，小写表示该符号是局部的（比如在变量声明时加上`static`关键字），大写表示符号是全局的。这里摘录`man nm`中的解释：

	> the symbol type: U (undefined), A(absolute), T (text section symbol), D (data section  symbol),  B  (bss section  symbol),  C  (common  symbol), S (symbol in a section other than those above), or  I  (indirect  symbol).   If the symbol is local (non-external), the symbol's type is instead represented  by  the  corresponding  lowercase letter.

3. 符号本身。

####链接

在`C`中可以通过`extern`关键字声明一个在其他`.c`文件中定义的变量或是函数。在上面`test.c`文件中加入一个`foo`函数的声明如下：

	int a = 2;
	int b;
	extern void foo();

	int main(int argc, char *argv[])
	{
	  return 0;
	}

如果这时直接编译`test.c`则会报错：

	Undefined symbols for architecture x86_64:
	  "_foo", referenced from:
	      _main in test-352996.o
	ld: symbol(s) not found for architecture x86_64
	clang: error: linker command failed with exit code 1 (use -v to see invocation)

其实这个错误就是由链接器报出的。因为`test.c`中声明了一个在外部定义的函数`foo`，但是我们只提供给链接器一个目标文件`test-352996.o`，在这个目标文件中链接器没有找到`foo`的定义。

我们增加一个`foo.c`文件：

	void foo() 
	{
	    int a = 0;
	}

然后再写一个makefile文件：

	final : foo.o test.o
		gcc -o final foo.o test.o
	foo.o : foo.c
		gcc -c foo.c
	test.o : test.c
		gcc -c test.c

执行`make`命令，我们可以得到三个ELF文件：两个可重定位目标文件`test.o`和`foo.o`，一个可执行文件`final`.

这时再用`nm`看一下`test.o`中的符号表:

	0000000000000028 D _a
	0000000000000004 C _b
	                 U _foo
	0000000000000000 T _main

可以看到符号`_foo`的类型是`U`，也即未定义。再看一下`foo.o`中的符号表:

	0000000000000000 T _foo

因此链接器可以在`foo.o`中找到符号`_foo`.

另外注意到`_main`和`_foo`这两个符号的值都为0，也就说明在被链接器链接之前这两个符号的虚拟内存地址是不确定的。那被链接之后它们的值会变成什么呢？我们再看一下可执行文件`final`的符号表：

	0000000100000000 T __mh_execute_header
	0000000100001000 D _a
	0000000100001004 S _b
	0000000100000f50 T _foo
	0000000100000f60 T _main

可以看到这时`_main`和`_foo`都有了确定的值。当然这两个地址也只是逻辑地址，真实的内存地址还取决于操作系统把这个可执行文件装载到内存的哪个位置了。