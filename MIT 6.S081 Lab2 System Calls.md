# MIT 6.S081 Lab2 System Calls

在上一个Lab中，我们通过已有的System Calls实现了一些用户程序。本次的lab要求我们亲自实现可以使用的System Calls，从而更加深刻地理解这些系统调用工作的方式。

## Using gdb

gdb是一个简单的调试工具，可以帮助我们更加深刻地跟踪程序在执行时，变量、语句的状态。我们可以在QEMU工具中使用gdb进行调试。我们将会在这个Lab中回答一些问题。

> Looking at the backtrace output, which function called `syscall`?

`kernel/trap.c`的`usertrap ()`函数调用了`syscall`。

> What is the value of `p->trapframe->a7` and what does that value represent? (Hint: look `user/initcode.S`, the first user program xv6 starts.)

不断前进直至完`num = p->trapframe->a7;   `后，在gdb输入` p p->trapframe->a7`，可以看到目标值的内容。查看其他的文件发现，a7寄存器中存放的7对应了系统调用`exec`的编号7。

```gdb
(gdb) p p->trapframe->a7
$2 = 7
```

> What was the previous mode that the CPU was in?

首先，我们可以打印`$sstatus`的内容，然后查阅[RISC-V privileged instructions](https://github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf)，通过观察寄存器值的标识位来判断CPU当前所处的状态。该寄存器不同比特位表示的意义如下图所示：

![21511697520984_.pic](https://raw.githubusercontent.com/Enqurance/Figures/main/202310171336718.jpg)

如果SPP位为0则CPU处于用户模式；如果是1则CPU处于管理员模式。通过gdb的调试命令，`$sstatus`的值为`1000000000000000000000000000100010`，可见其SPP位为0，表明CPU在进入系统调用前处于用户模式。

> Write down the assembly instruction the kernel is panicing at. Which register corresponds to the variable `num`?

按照指导书的要求，我们用`num = * (int *) 0;`代替`num = p->trapframe->a7;`，然后make。产生的部分报错信息如下：

```shell
scause 0x000000000000000d
sepc=0x0000000080002050 stval=0x0000000000000000
panic: kerneltrap
```

其中，`spec`告诉我们panic的地址。指导书提示我们`kernel.asm`是内核编译好的汇编文件，因此我们可以进入这个文件，按照`spec`提供的panic地址查找出错的位置。

```assembly
//  num = p->trapframe->a7;
  num = * (int *) 0;
    80002050:	00002683          	lw	a3,0(zero) # 0 <_entry-0x80000000>
```

通过汇编指令可知，`$a3`是与`num`有关的寄存器。

> Why does the kernel crash? Hint: look at figure 3-3 in the text; is address 0 mapped in the kernel address space? Is that confirmed by the value in `scause` above? (See description of `scause` in [RISC-V Privileged Instructions](https://pdos.csail.mit.edu/6.S081/2023/labs/n//github.com/riscv/riscv-isa-manual/releases/download/Priv-v1.12/riscv-privileged-20211203.pdf))

首先，我们通过`spec`知道指令执行发生错误的位置为`0x0000000080002050`，那么可以先在该位置创建一个断点：

```shell
(gdb)p *0x0000000080002050
```

随后前往该断电，执行`layout`命令查看当前汇编指令的执行情况：

```assembly
(gdb) layout asm

│B+>0x80002050 <syscall+20> lw      a3,0(zero) # 0x0                             │
│   0x80002054 <syscall+24> addiw   a4,a3,-1                                     │
│   0x80002058 <syscall+28> li      a5,20                                        │
│   0x8000205a <syscall+30> bltu    a5,a4,0x80002078 <syscall+60>  							 │
```

可以发现出错的指令。此时内核已经panic了，我们可以Ctrl+C打断gdb，按照指导书的提示，查看`$scause`寄存器的内容：

```shell
(gdb)p $scause
$1 = 13
```

`$scause`寄存器是一个SXLEN（32）位的读写寄存器，其格式如下图所示。当内核在管理者（S）模式进入陷阱时，导致陷阱的事件的编码会被写入该寄存器。此处，`$scause`寄存器的值是13，查看RISC-V手册，发现13对应的错误情况为**【Load page fault】**，即页加载错误。这说明0地址是一个错误的地址，不能在内核空间中访问，这和内核空间的虚拟地址是从`0x80000000`的特点是相符合的。

> What is the name of the binary that was running when the kernel paniced? What is its process id (`pid`)?

通过gdb，我们可以打印出错的进程的进程名称和进程id：

```shell
(gdb) p p->name
$1 = "initcode\000\000\000\000\000\000\000"
(gdb) p p->pid
$2 = 1
```

这能帮我们更好地跟踪错误发生的位置，以便于debug。

### Conclusion of gdb section

gdb是一个原始但是强大的调试工具，能够帮助我们定位程序中发证错误的位置、种类，帮助我们追踪bug。一些现代集成开发环境也提供了强大的调试工具供开发者使用，大大提升了软件开发的效率。

## System call tracing

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

这一Task要求我们实现一个名为`trace`的系统调用，使用格式为`trace [mask] [command]`。该系统调用携带一个掩码，随后是一条命令。该系统调用会执行附带的命令，并且若遇到和掩码对应的系统调用，则输出相应的信息。如`trace 32 ...`，而`sys_fork`的值为5，1按位左移5位即为32，因此该系统调用会在`[command]`使用`sys_fork`的时候输出信息。

该系统调用的源程序在实验代码中已经给出，我们需要在xv6中进行系统调用配置使之生效。

```C
int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
```

想要将一个新的系统调用加入到xv6中，需要修改好几个文件：

- 首先需要在`user.h`文件中声明该系统调用，使其能够在用户空间被使用：

  ```C
  // user/user.h
  ......
  int uptime(void);
  int trace(int);
  ......
  ```

- 随后，需要在`usys.pl`中注入该系统调用：

  ```c
  // user/usys.pl
  ......
  entry("uptime");
  entry("trace");
  ......
  ```

- 在`syscall.h`中为该系统调用分配一个编号，按顺序分配即可（当然，也可以随意分配一个之前没有使用个编号）

  ```C
  // kernel/syscall.h
  #define SYS_close  21
  #define SYS_trace  22
  ```

此时，操作系统就知道一个新的系统调用了，即系统调用号22对应的`trace`。然而，我们的任务还没有完成，此时我们还没有实现该系统调用的全部功能和用户接口。

- 在`trace`系统调用中，其在进程进行系统调用时，通过比较需要执行系统调用的值和当前掩码实现对掩码指定的系统调用的追踪。掩码是如何保存的？当我们使用`trace`时，跟随的掩码参数会保存在进程结构体的一个新的域中，因此我们需要在`proc.h`中开辟这么一个域：

  ```C
  // kernel/proc.h
  struct proc {
    ......
    uint trace_mask;              // Trace mask code
    ......
  };
  ```

- 在`sysproc.c`中实现该系统调用。我们需要获取命令行参数中的掩码，并且赋值给当前进程的`trace_mask`域。可以在`kernel/syscall.c`中查看用于获取命令行参数的相关函数的使用方法，并且调用`myproc()`获取当前进程。

  ```C
  // kernel/sysproc.c
  uint64
  sys_trace(void) {
      int trace_id;
      argint(0, &trace_id);		// 使用argint()获取参数
      myproc()->trace_mask = trace_id;		// 赋值给trace_mask
      return 0;
  }
  ```

  此时，在使用`trace`系统调用时，当前进程就可以获得该系统调用的掩码参数，并且保存在当前进程当中。

- 最后，我们需要在`syscall.c`中引入该系统调用函数，并修改`syscall()`函数，令其在当前进程掩码有效时能够打印掩码涵盖的系统调用的信息：

  ```C
  // kernel/syscall.c
  ......
  extern uint64 sys_trace(void);		// 引入trace
  ......
  static uint64 (*syscalls[])(void) = {		// 加入函数指针用于调用
    			......
          [SYS_trace]   sys_trace,
  };
  char *syscall_arr[23] = {			// 声明一个【索引-系统调用号】的映射函数
          "fork",
    			......
          "trace"};
  
  void
  syscall(void) {
  		......
      num = p->trapframe->a7;
      if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
  				......
          // trace信息
          if (p->trace_mask & (1 << num)) {
              printf("%d: syscall %s -> %d\n", p->pid, syscall_arr[num - 1], p->trapframe->a0);
          }
      } else { ...... }
  }
  ```

至此，`trace`系统调用就完成了。我们通过对一系列文件的修改，在内核中实现了系统调用的声明和实现，并且对用户暴露了该系统调用。

## Sysinfo

`sysinfo`的实现流程和`trace`的大致相似，只有极小的区别。该系统调用的功能是获得当前进程的一些信息，包括当前进程空闲内存的大小以及当前处于`UNUSED`状态的进程数量。该系统掉用将这些信息从内核拷贝到用户空间的一个`sysinfo`结构体。

- 首先需要在`user.h`文件中声明该系统调用，并在`usys.pl`中注入该系统调用，这些步骤和上一个Task一样：

  ```C
  // user/user.h
  struct sysinfo;
  ......
  int uptime(void);
  int trace(int);
  int sysinfo(struct sysinfo *);
  ......
  
  // user/usys.pl
  ......
  entry("uptime");
  entry("trace");
  entry("sysinfo");
  ......
  ```

- 随后在`sysproc.c`中实现该系统调用

  ```C
  uint64
  sys_sysinfo(void) {
      uint64 addr;
      struct sysinfo s_info;
      s_info.nproc = unused_proc();
      s_info.freemem = get_freemem();
      argaddr(0, &addr);
      // copy the struct back to user space
    	// 此处copyout()用于将指定地址的制定内容段拷贝到用户空间
    	// 用法参见sys_fstat()(kernel/sysfile.c)和filestat()(kernel/file.c)
      if (copyout(myproc()->pagetable, addr, (char *) &s_info, sizeof(s_info)) < 0)
          return -1;
      return 0;
  }
  ```

  其中，`unused_proc()`函数和`get_freemem()`函数需要自己实现（分别在`kernel/proc.c`和`kernel/kalloc.c`中），此处讲解一下实现思路。

  - `unused_proc()`返回一个整数，记录当前处于`UNUSED`状态的进程数量。按照指导书的指示，进入`proc.c`文件中，阅读该文件的代码和注释，我们会找到一些Hint。在内核中，所有的进程都以结构体的方式被管理，并记录在`proc[NPROC]`数组中：

    ```C
    struct proc proc[NPROC];		// 存储进程结构体的
    ```

    我们可以直接遍历该数组，查看其中进程结构体的`state`域，从而记录处于该状态进程的数量：

    ```c
    int unused_proc() {
        int unused = 0;
        for(int i = 0;i < NPROC;i++){
            if(proc[i].state != UNUSED) {
                unused++;
            }
        }
        return unused;
    }
    ```

  - `unused_proc()`返回一个整数，记录当前空闲内存的大小。在`kalloc.c`中，也可以找到一些提示。`run`和`keme`结构体为我们保存了必要的信息。xv6通过页表实现内存的页式管理，并将记录页表的结构体组织成链：

    ```C
    struct run {
      struct run *next;
    };
    
    struct {
      struct spinlock lock;
      struct run *freelist;		// 记录了第一个空闲页的指针
    } kmem;
    ```

    沿着`freelist`不断向前遍历直至链表尾端，即可获得空闲内存的大小。每一页的内存大小为`PGSIZE`字节：

    ```c
    int get_freemem(void){
      struct run *r;
      int cnt = 0;
      acquire(&kmem.lock);
      r = kmem.freelist;
      while(r) {
        cnt += PGSIZE;
        r = r->next;
      }
      release(&kmem.lock);
      return cnt;
    }
    ```

  有了这两个函数，结合拷贝函数，我们就可以实现`sysinfo`系统调用了。

- 最后，依旧是在`syscall.c`中注册该系统调用，此处不赘述。

在实现`sysinfo`系统调用的过程中，我们初窥xv6的进程管理和页表系统，引出了下一个lab的学习内容。