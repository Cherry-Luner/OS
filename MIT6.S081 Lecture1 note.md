

| MIT 6S.081 Lecture1 note |
| ------------------------ |



### OS课程目标

1. 抽象硬件，为上层提供**便利**和**可移植性**
2. 在应用程序中**多路复用硬件**
3. 使得应用程序具有**隔离性**包括错误(<u>本应用程序的bug不影响其他程序</u>)
4. 允许应用程序**协同工作**
5. 控制共享的**安全性**
6. 支持广泛的应用程序

### 全局视角

|    用户空间 如gcc编译器，vim编辑器，游戏，数据库    |
| :-------------------------------------------------: |
| **内核空间 为上层提供syscall，如open，write，read** |
|              **硬件 CPU Disk RAM Net**              |

### OS kernel 提供什么服务

1. process进程
2. 内存分配
3. 文件系统-------创建删除移动文件或目录
4. 访问控制
5. 其他诸如，网络，时间

### Simples

```c
// copy.c: copy input to output.

#include "kernel/types.h"
#include "user/user.h"
/*read第一个参数是文件描述符，指向一个之前打开的文件。Shell会确保默认情况下，当一个程序启动时，文件描述符0连接到console的输入，文件描述符1连接到了console的输出。所以我可以通过这个程序看到console打印我的输入。当然，这里的程序会预期文件描述符已经被Shell打开并设置好。这里的0，1文件描述符是非常普遍的Unix风格，许多的Unix系统都会从文件描述符0读取数据，然后向文件描述符1写入数据。

read的第二个参数是指向某段内存的指针，程序可以通过指针对应的地址读取内存中的数据，这里的指针就是代码中的buf参数。在代码第10行，程序在栈里面申请了64字节的内存，并将指针保存在buf中，这样read可以将数据保存在这64字节中。

read的第三个参数是代码想读取的最大长度，sizeof(buf)表示，最多读取64字节的数据，所以这里的read最多只能从连接到文件描述符0的设备，也就是console中，读取64字节的数据。*/
int
main()
{
  char buf[64];

  while(1){
    int n = read(0, buf, sizeof(buf));
    //read返回值可能是读取的字节数，read可能从一个文件读数据，如果到达了文件的结尾没有更多的内容了，read会返回0。如果出现了一些错误，比如文件描述符不存在，read或许会返回-1。
    if(n <= 0)
      break;
    write(1, buf, n);//向buf处写入n个字节
  }// 0 1 是标准文件描述符，0代表标准输入，1代表标准输出，与显示器直接相连(一般来说)，1与键盘直接相连(一般来说)

  exit(0);
}

运行结果：
    copy 
    asdsd
    asdsd
```

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"
/*
这个叫做open的程序，会创建一个叫做output.txt的新文件，并向它写入一些数据，最后退出。我们看不到任何输出，因为它只是向打开的文件中写入数据。
文件描述符本质上对应了内核中的一个表单数据。内核维护了每个运行进程的状态，内核会为每一个运行进程保存一个表单，表单的key是文件描述符。这个表单让内核知道，每个文件描述符对应的实际内容是什么。这里比较关键的点是，每个进程都有自己独立的文件描述符空间，所以如果运行了两个不同的程序，对应两个不同的进程，如果它们都打开一个文件，它们或许可以得到相同数字的文件描述符，但是因为内核为每个进程都维护了一个独立的文件描述符空间，这里相同数字的文件描述符可能会对应到不同的文件。
*/
int
main()
{
  int fd = open("output.txt", O_WRONLY | O_CREATE);
     //将文件名output.txt作为参数传入，第二个参数是一些标志位，用来告诉open系统调用在内核中的实现：我们将要创建并写入一个文件。open系统调用会返回一个新分配的文件描述符，这里的文件描述符是一个小的数字，可能是2，3，4或者其他的数字。
    
  write(fd, "ooo\n", 4);
	//文件描述符作为第一个参数被传到了write，write的第二个参数是数据的指针，第三个参数是要写入的字节数。数据被写入到了文件描述符对应的文件中。
  exit(0);
}
```

```c
// fork.c: create a new process

#include "kernel/types.h"
#include "user/user.h"
/*fork会拷贝当前进程的内存，并创建一个新的进程，这里的内存包含了进程的指令和数据。之后，我们就有了两个拥有完全一样内存的进程。fork系统调用在两个进程中都会返回，在原始的进程中，fork系统调用会返回大于0的整数，这个是新创建进程的ID。而在新创建的进程中，fork系统调用会返回0。所以即使两个进程的内存是完全一样的，我们还是可以通过fork的返回值区分旧进程和新进程。

当我们在Shell中运行东西的时候，Shell实际上会创建一个新的进程来运行你输入的每一个指令。所以，当我输入ls时，我们需要Shell通过fork创建一个进程来运行ls，这里需要某种方式来让这个新的进程来运行ls程序中的指令，加载名为ls的文件中的指令*/
int
main()
{
  int pid;

  pid = fork();

  printf("fork() returned %d\n", pid);

  if(pid == 0){
    printf("child\n");
  } else {
    printf("parent\n");
  }
  
  exit(0);
}
```

```c
// exec.c: replace a process with an executable file

#include "kernel/types.h"
#include "user/user.h"
/*
代码会执行exec系统调用，这个系统调用会从指定的文件中读取并加载指令，并替代当前调用进程的指令。从某种程度上来说，这样相当于丢弃了调用进程的内存，并开始执行新加载的指令。

exec系统调用会保留当前的文件描述符表单。所以任何在exec系统调用之前的文件描述符，例如0，1，2等。它们在新的程序中表示相同的东西。
通常来说exec系统调用不会返回，因为exec会完全替换当前进程的内存，相当于当前进程不复存在了，所以exec系统调用已经没有地方能返回了。除非出错了，比如没有echo这个指令，那么会返回并且报错
*/
int
main()
{
  char *argv[] = { "echo", "this", "is", "echo", 0 };

  exec("echo", argv);
   /*操作系统从名为echo的文件中加载指令到当前的进程中，并替换了当前进程的内存，之后开始执行这些新加载的指令。同时，可以传入命令行参数，exec允许你传入一个命令行参数的数组，这里就是一个C语言中的指针数组，在上面代码的第10行设置好了一个字符指针的数组，这里的字符指针本质就是一个字符串（string）。argv[]的结尾必须是空指针，用来确定结尾*/

  printf("exec failed!\n");

  exit(0);
}
运行结果：
    exec
    this is echo
```

```c
#include "kernel/types.h"
#include "user/user.h"

// forkexec.c: fork then exec

int
main()
{
  int pid, status;

  pid = fork();
  if(pid == 0){
    char *argv[] = { "echo", "THIS", "IS", "ECHO", 0 };
    exec("echo", argv);
    printf("exec failed!\n");
    exit(1);
  } else {
    printf("parent waiting\n");
    wait(&status);
    printf("the child exited with status %d\n", status);
  }
/*
Unix提供了一个wait系统调用。wait会等待之前创建的子进程退出。当我在命令行执行一个指令时，我们一般会希望Shell等待指令执行完成。所以wait系统调用，使得父进程可以等待任何一个子进程返回。这里wait的参数status，是一种让退出的子进程以一个整数（32bit的数据）的格式与等待的父进程通信方式。所以在第17行，exit的参数是1，操作系统会将1从退出的子进程传递到第20行，也就是等待的父进程处。&status，是将status对应的地址传递给内核，内核会向这个地址写入子进程向exit传入的参数。*/
  exit(0);
}
运行结果:
	forkexec
	This is echo
    the child exited with status 0
```

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

// redirect.c: run a command with output redirected

int
main()
{
  int pid;

  pid = fork();
  if(pid == 0){
    close(1);
	//close(1)的意义是，我们希望文件描述符1指向一个其他的位置。在子进程中，我们不想使用原本指向console输出的文件描述符1。
    open("output.txt", O_WRONLY|O_CREATE);
	//open一定会返回1，因为open会返回当前进程未使用的最小文件描述符序号。因为我们刚刚关闭了文件描述符1，而文件描述符0还对应着console的输入，所以open一定可以返回1。在代码第16行之后，文件描述符1与文件output.txt关联。
    char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };
    exec("echo", argv);
    printf("exec failed!\n");
    exit(1);
  } else {
    wait((int *) 0);
  }

  exit(0);
}
运行结果:
	redirect
    cat output.txt
    this is redirected echo
```

