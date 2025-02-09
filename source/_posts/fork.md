﻿---
title: Linux下的进程浅谈（一）
date: 2019年4月5日
mathjax: true
tags: 
	- Linux
	- 系统编程
categories: Linux系统编程
---
# **进程相关概念**
### **程序和进程**
程序，是指编译好的二进制文件，在磁盘上，不占用系统资源(cpu、内存、打开的文件、设备、锁....)
进程，是一个抽象的概念，与操作系统原理联系紧密。进程是活跃的程序，占用系统资源。在内存中执行。(程序运行起来，产生一个进程)
程序 → 剧本(纸)		进程 → 戏(舞台、演员、灯光、道具...)
同一个剧本可以在多个舞台同时上演。同样，同一个程序也可以加载为不同的进程(彼此之间互不影响)
如：同时开两个终端。各自都有一个bash但彼此ID不同。

### **并发**
并发，在操作系统中，一个时间段中有多个进程都处于已启动运行到运行完毕之间的状态。但，任一个时刻点上仍只有一个进程在运行。
例如，当下，我们使用计算机时可以边听音乐边聊天边上网。 若笼统的将他们均看做一个进程的话，为什么可以同时运行呢，因为并发。
![](/image/fork/图片1.png)
cpu会分成很多的时间时间碎片，以供所有的程序都可以分到一定得cpu时间，使得大家都可以占用一定时间的cpu；
cpu数着纳秒过日子的，所以在微观上程序还是一个一个进行运算；但在宏观上，他们是并发的，在用户层面感觉他是同时运行的;
<escape><!-- more --></escape>

# **进程控制块PCB**
## **基本概念**
我们知道，每个进程在内核中都有一个进程控制块（PCB）来维护进程相关的信息，Linux内核的进程控制块是task_struct结构体。
/usr/src/linux-headers-3.16.0-30/include/linux/sched.h文件中可以查看struct task_struct 结构体定义。其内部成员有很多，我们重点掌握以下部分即可：
* 进程id。系统中每个进程有唯一的id，在C语言中用pid_t类型表示，其实就是一个非负整数。
* 进程的状态，有就绪、运行、挂起、停止等状态。
* 进程切换时需要保存和恢复的一些CPU寄存器。
* 描述虚拟地址空间的信息。
* 描述控制终端的信息。
* 当前工作目录（Current Working Directory）。
* umask掩码。
* 文件描述符表，包含很多指向file结构体的指针。
* 和信号相关的信息。
* 用户id和组id。
* 会话（Session）和进程组。
* 进程可以使用的资源上限（Resource Limit）。
## **进程状态**
进程基本的状态有5种。分别为初始态，就绪态，运行态，挂起态与终止态。其中初始态为进程准备阶段，常与就绪态结合来看。
![](/image/fork/图片2.png)

# **进程控制**
## **fork函数**
创建一个子进程。
函数原型  pid_t fork(void);	 头文件 #include <unistd.h> 
失败返回-1；成功返回：① 父进程返回子进程的ID(非负)	②子进程返回 0 
pid_t类型表示进程ID，但为了表示-1，它是有符号整型。(0不是有效进程ID，init最小，为1)
	注意返回值，不是fork函数能返回两个值，而是fork后，fork函数变为两个，父子需【各自】返回一个。
>测试代码
```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    pid_t pid; // 定义返回值类型接受fork返回值
    printf("我是第一句话\n");
    pid = fork();
    if(pid == -1)// fork失败 返回值是-1
    {
        perror("fork error:");
    }
    else if(pid == 0)// 返回值为0代表是子进程
    {
        printf("我是子进程\n");
    }
    else//其他情况就是父进程  父进程返回的PID 就是子进程的PID
    {
        printf("我是父进程\n");
    }
    printf("我是第二句话\n");
    return 0;
}
```
>运行结果

![](/image/fork/图片3.png)
这个运行结果有很多需要解释的地方，我们来一点一点解释;
>1.父进程与子进程运行顺序的关系

从我们的程序来看，是先打印了父进程，然后打印了子进程；那是不是父进程一定先运行呢？答案是否定的，在组成原理中表示。任意不同的进程都是争夺资源，谁抢到谁先运行。所以可见，这个程序中，父进程和子进程也是竞争关系。谁先得到cpu谁先运行;(虽然理论是这样的，但是有人尝试了很多次在不加限制的情况下，运行了很多次这样的程序，发现有98%的概率父进程先运行);
>2.创建子进程之后，子进程是从main函数下面执行呢？还是当前到哪里就从哪里执行呢？

因为在我们在fork之前添加了一条打印("我是第一句话"),所以，如果子进程也是在来一遍的话，显然这句话应该被打印两遍，但是现在只打印了一遍。而后面的那句""我是第二句话"打印了两遍；所以我们可以知道；子进程是在哪里fork之后开始执行后面的代码；

>3.为啥显示这么奇怪呢？

这个是因为shell监听的是主进程，然后此程序主进程先运行完毕，所以当主进程运行完毕之后，shell切换到前台，所以子进程打印的消息就在终端提示之后了;

## **循环创建多个子进程**
我们先写一个假的循环创建多个子进程的程序;
>循环创建3个子进程
```c

#include <stdio.h>
#include <unistd.h>

int main()
{
    pid_t pid; // 定义返回值类型接受fork返回值
    int i;
    for(i = 0; i < 3; i++)
    {
        pid = fork();
        if(pid == -1)// fork失败
        {
            perror("fork error:");
        }
        else if(pid == 0)// 子进程
        {
            printf("我是第%d个子进程\n", i + 1);
        }
        else
        {
            printf("我是父进程\n");
        }
    }
    return 0;
}
```
>运行结果

![](/image/fork/图片4.png)
是不是和我们的预期不太一样? 为什么呢？ 其实仔细想想也不难理解；因为子进程还在for循环中，他也会作为父进程创建子进程，所以会打印这么多的我是父进程和我是第几个子进程;
我们稍微画一下这个过程
![](/image/fork/图片5.png)
我只画了i到1， 其实我们的程序还有一层。我们可以发现，一共生成了2的n次方-1个子进程。那么我们如何避免这个尴尬呢？其实也不难；
>循环创建多个子进程
```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    pid_t pid; // 定义返回值类型接受fork返回值
    int i;
    for(i = 0; i < 3; i++)
    {
        pid = fork();
        if(pid == -1)// fork失败
        {
            perror("fork error:");
        }
        else if(pid == 0)// 子进程
        {
            break;// 子进程直接跳出循环
        }
        else
        {
            //父进程啥都不做  只创建子进程
        }
    }
    if(i < 3)// 代表这是子进程跳出循环后
    {
        printf("我是第%d个子进程\n", i + 1);
    }
    if(i == 3)// for循环终止后 i = 3 此时这个是父进程
    {
        printf("我是父进程\n");
    }
    return 0;
}
```
>运行结果

![](/image/fork/图片6.png)
我们可以看到，此时程序就是正确的了；
## **进程常用函数**
### getpid函数
>获取当前进程ID
    pid_t getpid(void);	

### getppid函数
>获取当前进程的父进程ID
	pid_t getppid(void);

### getuid函数
>获取当前进程实际用户ID
    uid_t getuid(void);
获取当前进程有效用户ID
    uid_t geteuid(void);

### getgid函数
>获取当前进程使用用户组ID
    gid_t getgid(void);
获取当前进程有效用户组ID
    gid_t getegid(void);