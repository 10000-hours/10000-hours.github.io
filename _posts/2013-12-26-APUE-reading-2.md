---
layout: post
title: APUE读书笔记2
category: detail
tags: APUE Unix
---


Chap.8 Process Control
========================

PID通常是个整数，并且是唯一的。

创建一个进程可以在父进程中使用fork，fork会给子进程返回0（成功），
给父进程返回子进程PID（成功），-1（失败），fork的子进程会“共享”
父进程的一些资源，具体为：

1. **拷贝** 父进程的data space, heap, stack
2. **共享** 父进程的text segment

在实现fork上，可以使用copy-on-write（COW）机制来提高效率（只有当写时，
才拷贝相关的数据）

对于父进程和folk生成的子进程，其具体的调度（执行）时间是不确定的，与kernel
的调度实现是相关的，所以会涉及race condition情况。

folk的子进程会"duplicate"所有父进程的FD，父进程和子进程共享相同的file table
entry，以及相同的file offset，也就是说，如果父进程folk一个子进程，并且等待
子进程完成，如果在等待过程中子进程向某个共享的文件写内容（file offset更改）
那么，等子进程完成，返回父进程是，相应的file offset也会更新。

通常使用folk的原因（场景）：

1. 当一个进程想duplicate自己，父进程和子进程可以同时执行代码段的不同部分，
   例如，web server，当请求来时，父进程call folk，让folk的子进程来处理这个
   请求，父进程继续等待下一个请求。（父进程调度，子进程完成业务）
2. 当进程想执行另外的程序，通常在进程执行folk后，继续执行exec(某些操作系统，
   会将folk和exec两个操作合为一个spawn，来完成此目的）

无论子进程是否正常返回（正常:exit,_exit,_Exit从子进程返回，异常：signal, abort等
从kernel返回），父进程总是可以得到子进程结束的状态码，但是如果父进程比子进程
更早结束，怎么办？答案是，这个子进程的父进程会自动变为init进程（Pid=1),这样
保证每个进程都有子进程。

当一个进程结束（正常或异常），kernel都会通过向其父进程发送一个异步的SIGCHLD信号，
来通知父进程其子进程结束。

调用wait/waitpid的进程会：

1. 如果所以的子进程都在运行，则会block
2. 如果子进程结束并在等待结束状态码，父进程会立即返回（带有子进程的结束状态码）
3. 如果没有子进程了，立即返回（with an error)


exec，当一个进程调用exec时，整个进程会被被调用的程序替代(exec会调用某个程序），这个
被调用的程序开始执行自己的main函数。exec并不会更改PID（没有新进程创建），exec只是替换
当前的进程（text, data, heap, stack, 也就是一个disk上的全新程序）

system函数，可以执行参数（字符串）指向的程序，使用fork, exec, waitpid实现。


Chap.9 Process Relationships
===============================

login分为local/remote，一个典型的过程是：

1. kernel会创建init进程（PID=1）
2. init进程读取/etc/ttys(每行一个terminal设备），为每个terminal设备folk自己，并且exec getty
3. getty会调用open来打开这个terminal设备,一旦设备打开,stdin/stdout/stderr会设定
4. getty会提示login:来要求用户输入用户名
5. getty会在用户输入用户名后结束，并exec /bin/login(gettytab提供相应的初始变量）
6. login会调用getpwnam来获取password file entry
7. login调用getpass来提示用户输入密码（password:)
8. 获取密码后，调用crypt来加密，并且与shadow file中的密码进行比对，来确认是否可以login
9. 如果Login成功，则根据password file entry中的bash/home/uid/gid等进行相应的环境初始化操作


如果是remote的login，则会涉及到pseudo terminal的概念。

job control主要是指foreground/background jobs，而foreground/background会接受不同的信号，以及
不同的表现（对于stdio)

A process belongs to a process group, and the process group belongs to a session. The session may or may not have a controlling terminal. If the session does have a controlling terminal, then the terminal device knows the process group ID of the foreground process.

Chap. 10 Signals
==================

signal disposition指的是进程对于某个信号的处置（signal callback function)，
对于exec的进程，其signal disposition会改变，因为父进程的处置函数的地址对于子进程没有意义。
而folk的进程，子进程会继承父进程的信号处置。

reentrant functions,对于一个进程，安装有特定signal的signal handler，当signal触发时，当前的进程
执行就会暂停，转而执行handler，当handler返回（无exit,longjmp)时，进程继续执行。因为signal的触发
时机是不确定的，所以可能在malloc时signal触发开始执行handler，而出现问题。reentrant function就是
为了避免这个问题的，reentrant function,也称作aysnc-signal safe，函数不会引起不一致。

abort函数会想进程发送一个无法忽略的SIGABRT信号, ISO C规定abort函数会调用raise(SIGABRT)

Chap. 11 Threads
==================

pthreads代表"POSIX threads".

Thread ID, 对应于Process ID, pthread_t类型，可以使用pthread_equal来比较, pthread_t通常实现时
可以是个无符号整形，当然也可是别的结构体，所以为了portable，不能默认Thread ID是整形，而应该
使用pthread_t类型。Thread ID不像Process ID必须是唯一的，Thread ID只需要在特定的进程里唯一
即可。

thread可以通过pthread_create来创建, 当一个线程创建成功，此线程的执行时间，或者创建此线程的线程
执行时间，二者的先后顺序是不确定的。

线程终止，进程中的任一线程调用exit, _Exit, _exit，则整个进程终止；类似，如果某个线程受到终止的
信号，则整个进程终止。

不影响其它线程的终止包括：

1. 从开始过程中返回（start routine)，返回值是此线程的exit code
2. 线程可以被同一进程中的其它线程取消
3. 线程可以调用pthread_exit

pthread_join等待进程执行结束（block直至结束），pthread_cancel取消同进程的其它线程，不会
等待结束，只是发出请求。

线程也可以注册一些cleanup handler，会在结束前执行，类似于exitat，执行顺序与注册顺序
相反（stack).

线程和进程有类似的一些方法，具体如下：

![thread_process](/assets/images/thread_process.png)

线程也可以detach, pthread_detch，detch的线程不能再join(join的结果undefined)

线程同步：对于只读数据，或者无共享数据的进程，无须线程同步，但通常共享的数据（资源）是多个线程可读可写
的，所以需要同步。

mutex可以用来同步，mutex相当于锁，pthread_mutex_lock/pthread_mutex_unlock等可以用于这些同步过程。
mutext可能会出现deadlock，例如线程A为资源a加锁，请求b资源，线程B为b加锁，请求a资源，于是A和B永远无法
继续，一直死锁。可以通过所有线程对于资源的加锁顺序相同来避免此问题。

reader-writer lock，也叫做shared-exclusive lock，粒度更小，对于资源的读写锁，可以有不同的机制，适合
heavy read的程序。

condition variable，条件变量也可用于同步。

Spin locks，也可用于同步，不过是更加low level的lock.

Barriers,也是另外的一种同步机制，barrier允许每个进程wait直到所有合作的线程执行到相同的point，然后
再继续执行。pthread_join就是barrier的特例。

Chap. 12 Thread Control
=========================

当我们使用pthread_create创建线程时，可以传递参数来控制新创建线程的默认属性，从而在更小粒度上
控制线程的属性和行为。

同步控制方面的mutex, rw lock, condition variable, spin lock都有可以设置的属性，来控制同步的具体
行为。

线程和信号（thread and signal)，进程的信号系统已经很复杂，线程的信号更加复杂。一个进程中的线程，共享
同一个信号handler，那么信号触发时，进程中哪个线程来处理呢？1. 对于hardware fault(例如除数为0），则由
引起signal的线程处理（执行handler) 2. 其它信号，则由进程中的任一线程处理。一旦进程中的一个线程修改了
handler，同进程其它线程的handler一并修改。

进程中的线程Hold相同的FDs, 所以可能存在读取不一致的问题（如线程A seek, B seek, A read, B read，导致
A读的结果与B相同，而与预期不同），所以需要使用原子操作的pread/pwrite来完成。

Chap. 13 Daemon Processes
=========================

单实例的daemon程序通常使用lockfile来保证单一实例，通常有如下惯常使用：

1. lock file一般位于/var/run/name.pid 此文件的内容通常是daemon进程的PID
2. 如果daemon支持配置选项，通常配置位于/etc/name.conf
3. daemon通常在系统启动是启动，启动脚本通常位于/etc/rc* or /etc/init.d/*,
   当然也可以在命令行下启动。
4. 通常daemon的配置只在启动时读取，所以当配置更改时，通常需要restart这个
   daemon,也有程序支持SIGHUP信号来重新读取配置。

Daemon通常也使用在Client/Server模型中，Daemon程序充当Server角色，等待Client
的连接和通信。


Chap. 14 Advanced I/O
========================

Nonblocking I/O

对于'slow' I/O，一个read/write可能会永久地block（因为IO没有满足相应的条件），
Nonblocking I/O则在没有满足条件时，可以及时退出，从而避免block.

可以通过打开文件时设置file flag为O_NONBLOCK来设置I/O为noblocking,或者更改
已经打开的文件FD的flag为O_NONBLOCK来避免block.

也可以通过multiple thread来避免使用noblocking IO，例如一个thread block等待file
其它thread可以继续工作,但是通常multiple thread的同步带来的复杂性可能会得不偿失。

Record Locking

有些应用需要防止同一时刻有其它进程来修改同一record，所以需要有机制来阻止其它进程
的写，称为record locking，这里的record是byte-range,也就是一个文件的部分，或者
整个文件。例如DB程序，通常写时要阻止其它进程的写，以避免不一致性。

Record Locking是byte级的，在加锁时可能会出现deadlock。通常使用fcntl来获取锁。

锁继承与释放规则：1）locks是与process和file关联的，也就是说，当一个process退出时，
所以其持有的locks将会释放；当一个FD关闭时，施加其上的所有进程持有的锁，也会释放。

例如 

    fd1=open(pathname, ...); 
    read_lock(fd1, ...);
    fd2 = dup(fd1);
    close(fd2);  //此时fd1上的锁也是会释放
2) fork的子进程不会继承父进程的锁，子进程可以通过fcntl来申请自己的锁。
3）通过exec的新程序会继承锁，但是如果close-on-exec flag设置在某个FD上，
   FD上的所有锁会释放，当exec的某个程序关闭此FD时。

I/O multiplexing

对于多个FD的IO，我们不知道何时IO的数据ready了，可以进行IO了，当然可以通过轮询（polling)
来定期的查询，缺点是：间隔时间，消耗CPU。

于是有了I/O multiplexing机制，通常有select/pselect/poll方法.

在底层，asynchronous I/O也是asynchronous，而不是polling，例如，一个读操作，OS会调用read
从而形成一系列的磁盘操作，这些操作会写在特定的寄存器或者内存位置，并且磁盘控制器被告知
有pending的操作，此时OS会忙其它的事务，等到磁盘控制器完成这个pending的操作时，才会向OS发出
中断(interrupt)，从而OS完成剩余的读操作，继而返回给调用的程序。

vectored I/O, 也称作scatter/gather I/O，一个调用可以将多个buffer写到一个data stream中，或
将一个stream中的数据读取到多个buffer中，通常使用writev/readv.

例如一个场景，两个buffer，写到一个文件中，可能的办法：1）写2次 2）临时生产一个大的buffer,
将两个buffer拼接，然后写1次 3）使用writev来写一次。实际测试时，通常2）3）有更好的效率（更少
的系统调用）。

Memory Mapped I/O

将一个buffer映射为disk上文件的一个range，当我们读取buffer时，相应的文件的bytes会被读取，当我
们向buffer中写时，相应文件会被写入。mmap函数常用于此，其一个参数startaddr通常是system page
size的整数倍。


Chap. 15 Interprocess Communication
======================================

常用的有：

1. Pipe
2. FIFO(named Pipes)
3. unix domain sockets
4. message queue
5. semaphores
6. shared memory

Pipe: 通常是单向的（一个方向），int pipe(int fd[2]), 其中fd[0]是用于读，fd[1]用于写。
为了简化，通常有popen/pclose两个函数来供使用。

FIFO, 也称作named pipe, 可用于无关的进程共享数据，通常也用于Client/Server这种结构。

message queue/semaphores/shared memory称作XSI IPC，IPC Structure是由一个非负整数来
引用的，有几个不足：1）没有reference count，也就是说一旦创建一直存在，除非某个进程
显式删除或者系统重启 2）不是以file的形式存在，所以不支持ls/rm等

message queue是在kernel中创建的一个linked list，msgget可以获取或者创建（如果不存在）
一个queue.

semaphores用于有限资源的访问控制。

POSIX semaphores和XSI semaphores不同，我们应该使用POSIX.
