| MIT 6.S081 Lab2 system calls |
| ---------------------------- |

### System  call  tracing :

在本作业中，您将添加一个系统调用跟踪功能，该功能可能会在以后调试实验时对您有所帮助。您将创建一个新的`trace`系统调用来控制跟踪。它应该有一个参数，这个参数是一个整数“掩码”（mask），它的比特位指定要跟踪的系统调用。例如，要跟踪`fork`系统调用，程序调用`trace(1 << SYS_fork)`，其中`SYS_fork`是**kernel/syscall.h**中的系统调用编号。如果在掩码中设置了系统调用的编号，则必须修改`xv6`内核，以便在每个系统调用即将返回时打印出一行。该行应该包含进程id、系统调用的名称和返回值；您不需要打印系统调用参数。`trace`系统调用应启用对调用它的进程及其随后派生的任何子进程的跟踪，但不应影响其他进程。	



**提示：**

- 在**Makefile**的**UPROGS**中添加`$U/_trace`
- 运行**make qemu**，您将看到编译器无法编译**user/trace.c**，因为系统调用的用户空间存根还不存在：将系统调用的原型添加到**user/user.h**，存根添加到**user/usys.pl**，以及将系统调用编号添加到**kernel/syscall.h**，**Makefile**调用**perl**脚本**user/usys.pl**，它生成实际的系统调用存根**user/usys.S**，这个文件中的汇编代码使用RISC-V的`ecall`指令转换到内核。一旦修复了编译问题（注：如果编译还未通过，尝试先**make** **clean**，再执行**make** **qemu**），就运行`trace 32 grep hello README`；但由于您还没有在内核中实现系统调用，执行将失败。
- 在**kernel/sysproc.c**中添加一个`sys_trace()`函数，它通过将参数保存到proc结构体（请参见**kernel/proc.h**）里的一个新变量中来实现新的系统调用。从用户空间检索系统调用参数的函数在**kernel/syscall.c**中，您可以在**kernel/sysproc.c**中看到它们的使用示例。
- 修改`fork()`（请参阅**kernel/proc.c**）将跟踪掩码从父进程复制到子进程。
- 修改`kernel/syscall.c`中的`syscall()`函数以打印跟踪输出。您将需要添加一个系统调用名称数组以建立索引。

------



**首先呢，要搞定系统调用，就是你的操作系统里得认可有`trace`这个东西，虽然我们还没去实现它，但是得先声明一下有这么个玩意**

**怎么声明呢？**

1. 在**Makefile**的**UPROGS**中添加`$U/_trace`

   ​																	 ![image-20221118014054235](E:\Typore-Picture\image-20221118014054235.png)

2. 在`user.h`中添加系统调用原型

   <img src="E:\Typore-Picture\image-20221118014301630.png" alt="image-20221118014301630" style="zoom:80%;" />

3. 存根添加到**user/usys.pl**

   ![image-20221118014558481](E:\Typore-Picture\image-20221118014558481.png)

4. 系统调用编号添加到**kernel/syscall.h**

   ![image-20221118014743506](E:\Typore-Picture\image-20221118014743506.png)

5. 在现在这一步你已经向`xv6`说，这里有个`trace`的系统调用了，当你`make qemu`在运行`trace`时，依然会执行失败，因为你是声明了这个东西存在，但是你还没有实现呀

------

**实现**

​		在**kernel/sysproc.c**中添加一个`sys_trace()`函数,它通过将参数保存到`proc`结构体（请参见**kernel/proc.h**）里的一个新变量中来实现新的系统调用。

![image-20221118015819559](E:\Typore-Picture\image-20221118015819559.png)

​		我们要对这个函数函数做什么呢？

​				这个函数要将参数保存到`proc`结构体里面的一个新变量来**实现系统调用**，**所以我们要在这个结构体中增加一个新变量来放入我们的参数**

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  int mask;//新增加的变量 trace
};
```

​		

```c
/*我们现在处于内核之中，用户传进来的参数是没有办法直接就作用于sys_trace()的，从hints我们得知
从用户空间检索系统调用参数的函数在**kernel/syscall.c**中，您可以在**kernel/sysproc.c**中看到它们的使用示例。

我们知道我们的tance函数只需要一个整数，所以我们去查看，我们会发现
// Fetch the nth 32-bit system call argument.抓取32位的系统调用参数
int
argint(int n, int *ip)
{
  *ip = argraw(n);
  return 0;
}
我们一个个函数找下去，并且查看示例代码就会发现这个函数完美契合我们的需求
*/
uint64
sys_trace(void){
  int mask;	//参数哦
  if(argint(0,&mask)<0)
    return -1;
  myproc()->mask=mask;
  return 0;
}

//接下来去完成fork(),添加一行代码用来复制mask
np->mask=p->mask;

```

<img src="E:\Typore-Picture\image-20221118021754826.png" alt="image-20221118021754826" style="zoom:80%;" />

最后修改`syscall`函数，最后进行一个统一的过程

```c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_trace]   sys_trace,
[SYS_sysinfo] sys_info,c
};
```

|                            用户态                            |                            内核态                            |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| 调用`trace(32)`,参数放在寄存器`a0`，系统调用号放在`a7`，`syscalls`[`a7`]，来调用sys_trace()，现在开始陷入内核 |                                                              |
|                                                              | 去执行我们实现了的内核代码，sys_trace()函数，`syscall`将其返回值记录在`p->trapframe->a0` |
|                          回到用户态                          |                                                              |

注意 : 只有进入`sys_trace()`，才能读取到参数呀，我们的代码就是那么写的呀~

​			而且也只有执行完了，我们拿到mask了才能确认是不是我们要跟踪的系统调用，我们在`sys_trace()`做的一切就是在实现内核



## Sysinfo :

​		老样子声明系统调用嗷~

​		`sysinfo`需要将一个`struct sysinfo`复制回用户空间；请参阅`sys_fstat()(kernel/sysfile.c*)`和`filestat()(**kernel/file.c***)`以获取如何使用`copyout()`执行此操作的示例。

​		没什么好说的，就是看代码，直到看懂了调用`copyout()`的使用方法为止

```c
//要获取空闲内存量，请在kernel/kalloc.c中添加一个函数
//要获取进程数，请在kernel/proc.c中添加一个函数
uint64
get_freemem(void){//去看klloc.c函数的分配，因为分配必须是去找空闲页面的
 struct run *r;
  acquire(&kmem.lock);
  r = kmem.freelist;
  int num = 0;
  while(r)
  {
    ++num;
    r = r->next;
  }
  release(&kmem.lock);
  return num * PGSIZE;
}
int  //在proc.c上面有个proc的数组，里面有所有的进程
getprocess(void){
  int num=0;
  struct proc *p;
  for(p = proc; p < &proc[NPROC]; p++) {
    if(p->state == UNUSED) {
      num++;
    } 
  }
  return num;
}

int 
getprocess(void){
  int num=0;
  struct proc *p;
  for(p = proc; p < &proc[NPROC]; p++) {
    if(p->state == UNUSED) {
      num++;
    } 
  }
  return num;
}

uint64
sys_info(void){
  uint64 st;
  if(argaddr(0, &st) < 0)
    return -1;
  struct proc *p = myproc();
  struct sysinfo info;
  info.freemem=get_freemem();
  info.nproc=getprocess();
  if(copyout(p->pagetable, st, (char *)&info, sizeof(info)) < 0)
      return -1;
  return 0;
}
/*要注意，因为是从内核复制到用户空间的，所以sys_info()创建一个info就好了，然后其他的函数给这个info赋值
	不要忘记了要声明一下结构体，以及在dfs.h中声明一下函数以免无法使用*/
```

