# 线程

[toc]

## 线程概述

线程是允许应用程序**并发**执行多个任务的一种机制。

同一个程序中的所有线程均会独立执行相同程序，且共享同一份全局内存区域，包括 初始化数据段（`.bss`），未初始化数据段，以及堆内存段。

传统意义上 UNIX 进程是多线程程序的一个特例，该进程只包含一个线程。线程是轻量级的进程（LWP），在 Linux 环境下线程的本质仍是进程。

进程是 **CPU 分配资源**的最小单位， 线程是**操作系统调度执行**的最小单位。

通过 `ps -Lf PID` 查看进程的所有线程

- 线程与进程的区别

  - 进程间信息难以共享。除去只读代码段外，父子进程并未共享内存，因此必须采用一些 IPC 进行信息交换
  - `fork()` 会复制内存页表和文件描述符等进程属性，调用在时间上的开销不菲
  - 线程能方便共享信息。只需将数据复制到共享（全局或堆）变量中即可
  - 创建线程的比进程快 10 倍甚至更多。线程共享虚拟地址空间

- 线程中的共享资源（内核中的数据）

  - PID, PPID
  - PGID, SID
  - UID, UGID
  - 文件描述符表
  - 信号处置
  - 文件系统的相关信息：文件权限掩码 umask，cwd
  - 虚拟地址空间（除栈, `.text`外）

- 线程中的非共享资源

  - 线程 ID
  - 信号掩码（阻塞信号集）
  - 线程特有数据

  - error 变量
  - 实时调度策略和优先级
  - 栈、本地变量和函数的调用链接信息

## 线程操作	

### 创建线程

`main()` 创建的线程成为 主线程， 其余为 子线程

```c++
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg);

	创建一个子线程
        thread 			: 传出参数，线程id
		attr 			: 设置线程的属性，一般使用NULL
		start_routine 	: 子线程需要处理的代码	
		arg				: start_routine 的参数
	成功返回 0
    失败返回错误号，和errno的含义不同，不能通过perror输出
         调用 char* strerror(errnum) （string.h）获取错误信息
```

*编译时要 加上 `-pthread ` 链接动态库*

### 终止线程

```c++
#include <pthread.h>

void pthread_exit(void *retval);
	终止线程，需要在待终止线程中调用
        retval : 返回值，可以在 pthread_join 中获取
```

*`main()` 线程退出不影响子线程的运行，因为在 return 0; 之前，意味着进程没有结束*

### 线程回收

```c++
#include <pthread.h> 

int pthread_join(pthread_t thread, void **retval);
	和一个已经终止的线程进行连接，回收子线程的资源
        retval : 接收子线程退出时的返回值
```

阻塞函数，调用一次只能回收一个子线程

一般在主线程中使用

- 传递二级指针的原因
  - 需要改变 `retval(void *)` 的值

### 线程分离

```c++
#include <pthread.h>

int pthread_detach(pthread_t thread);
	分离一个线程，被分离线程在终止时会自动释放资源返回给系统
```

非阻塞

- 不能多次分离，会产生不可预料的行为
- 不能 `join` 一个已经分离的线程

### 线程取消

```c++
#include <pthread.h>

int pthread_cancel(pthread_t thread);
	取消线程（终止线程），并不是立刻终止
```

只有线程执行到 取消点（系统定义的一些函数 `man pthreads 查看`） 后才能取消

- 理解为 当有 用户区到内核区 的切换时 到达取消点

### 其他操作

- `pthread_self()` 获取当前线程ID

- `int pthread_equal(pthread_t t1, pthread_t t2)`
  - 因为 `pthread_t` 在不同系统中实现不同，有的是用结构体实现（极少部分）

