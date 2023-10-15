# MIT 6.S081 Chapter1

本章节主要介绍操作系统的接口，旨在让我们观察并理解操作系统为用户提供的接口，并且让我们手动编写一些接口体会操作系统提供服务的过程。

### 1.1 进程和内存

进程是操作系统中的重要概念。只要接触过C语言，就知道可以使用`fork()`函数来进行进程创建。在xv6中，`fork()`函数会创建一份**父进程的副本**（内存空间、代码等）并交给机器执行，我们可以通过`fork()`函数的返回值来区别父子进程，父进程中`fork()`函数的返回值是子进程的id，而子进程中返回值则是0。下面的程序是一个例子。

```c
int pid = fork();
if(pid > 0){
  // 这个分支的内容在父进程中执行
	printf("parent: child=%d\n", pid);
	pid = wait((int *) 0);
	printf("child %d is done\n", pid);
} else if(pid == 0){
  // 这个分支的内容在子进程中执行
	printf("child: exiting\n");
	exit(0);
} else {
	printf("fork error\n");
}

// Output
parent: child=1234
child: exiting
parent: child 1234 is done
```

请注意，输出的顺序可能是混乱的，但是你一定可以排序出上面的输出内容。

接下来要介绍的是`exec`系统调用，即操作系统为用户提供的服务接口。`exec`系统调用会直接取代调用其进程的内存空间，并从指定的文件（包含要执行的指令）写入一片新的内存空间。如果`exec`成功执行，那么进程会直接结束，不再回到原来调用`exec`的程序；反之，则会回到原来调用`exec`的程序并且返回错误代码。一个调用该系统调用的例程如下：

```C
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;	// 用于标记数组的结束
exec("/bin/echo", argv);	// 第一个参数用于告知执行文件，第二个参数告知执行命令以及参数
printf("exec error\n");
```

其余的一些系统调用，在`user/sh.c`下能够看到，该文件包含了命令行界面实现的部分代码。该文件的149行是命令行交互程序的入口：

```C
int
main(void)
{
  static char buf[100];
  int fd;

  // Ensure that three file descriptors are open.
  while((fd = open("console", O_RDWR)) >= 0){
    if(fd >= 3){
      close(fd);
      break;
    }
  }

  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Chdir must be called by the parent, not the child.
      // 处理cd命令，打开文件描述符
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        fprintf(2, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      // 其他命令，创建子进程，并且执行命令行传入的命令
      runcmd(parsecmd(buf));
    wait(0);
  }
  exit(0);
}
```

### 1.2 I/O和文件描述符

文件描述符是一个较小的整数，表示进程可以读取的内核管理对象，**每一个进程都维护一组自己的文件描述符**。使用文件描述符，我们可以建立文件的索引，以便于我们快速找到并对文件进行操作。当我们打开新的文件时，系统会为该文件分配文件描述符，通常选取目前未被使用的最小整数。在xv6中，文件描述符0代表标准输入，1代表标准输出，2代表错误输出（当然在Linux中也是这样）。

我们可以使用`close()`系统调用关闭一个指定的文件描述符，也可以使用`open()`系统调用打开一个文件。使用`fork()`函数和`close()`、`open()`系统调用，我们可以方便地实现I/O重定向。

```C
char *argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
	close(0);
	open("input.txt", O_RDONLY);
	exec("cat", argv);
}
```

上述代码中，在创建子进程后，子进程使用`close()`关闭了文件描述符0，即关闭了标准输入的文件描述符。随后使用`open()`系统调用打开了文件`input.txt`。该系统调用把当前空闲的最小文件描述符0分配给了`input.txt`，**实现了标准输入的重定向**。除了文件名之外，我们还可以看到`open()`系统调用的可选参数，如`O_RDONLY`表示以只读方式打开`input.txt`；除此之外，还有`O_WRONLY`、`O_RDWR`等打开选项。

由上面的代码，我们可以看出`fork()`和`exce()`函数能够让我们通过一个子进程实现I/O重定向，并且不会影响原进程的I/O。不过需要注意的是，在使用`fork()`创建子进程时，对于同一个文件描述符位置的偏移是共享的。除此之外，我们也可以使用`dup()`系统调用拷贝文件描述符：

```C
fd = dup(1);
```

在xv6中，通过`fork()`拷贝的文件描述符和通过`dup()`拷贝的文件描述符共享便宜，其他情况则不然（使用`open()`打开文件等）。

文件描述符为进程提供了强大的抽象，将进程和需要读写的对象进行了隔离。

### 1.3 管道

管道是一对文件描述符，向进程暴露一片小内核缓冲区，其中一个文件描述符用于读，另外一个用于写。向管道的一端写入数据，就可以从读段读取数据，因此管道可以用于进程之间的通信。

下面的程序给出了一个管道的使用例子。首先，程序使用系统调用`pipe()`创建了两个文件描述符，用于管道的两端。随后，不同的进程开始不同的操作：

- 子进程：关闭标准输入对应的描述符0，使用`dup()`系统调用复制管道的读端，随后关闭子进程维护的数组`p`的文件描述符。子进程成功将标准I/O切换到了管道的读端（即文件描述符0索引到管道）
- 父进程：关闭自身管理的管道输入端对应的文件描述符`p[0]`，并且通过`write()`系统调用写入一段字符，随后关闭`p[1]`，即管道的写入端

```C
	int p[2];
	char *argv[2];
	argv[0] = "wc";
	argv[1] = 0;
	pipe(p);
	if(fork() == 0) {
		close(0);
		dup(p[0]);
		close(p[0]);
		close(p[1]);
		exec("/bin/wc", argv);
	} else {
		close(p[0]);
		write(p[1], "hello world\n", 12);
		close(p[1]);
	}
```

上面的代码向我们展示了进程是如何通过pipe来实现通信的。管道的读端会在缓冲区没有数据时进行等待，如果同时管道的写端都关闭了，那么管道的读端也会关闭。

`sh.c`文件中实现了一个`PIPE`功能，用于管道在命令行中的使用。`|`左侧的命令可以用于写，并重定向为右侧明亮的输入。详细内容可参考该文件。

### 1.4 文件系统

xv6提供管理字节数据的文件和目录系统，并且提供系统调用让用户操作文件和目录。

```c
mkdir("/dir");
fd = open("/dir/file", O_CREATE|O_WRONLY);
close(fd);
mknod("/console", 1, 1);
```

上面的代码给出了一个使用文件系统系统调用的案例。其中，`mknod()`系统调用打开了一指向设备的文件，当进程随后打开设备文件时，内核会将读写系统调用转移到内核设备实现，而不是传递给文件系统。

文件名和文件本身存在区别，文件本身被抽象为一个索引节点。索引节点可以拥有多个名字，即链接（links），每个链接包含文件的名称和对该文件索引节点的引用。在xv6中，索引节点是一个如下的数据结构：

```c
// stat.h
#define T_DIR 1 // Directory
#define T_FILE 2 // File
#define T_DEVICE 3 // Device

struct stat {
	int dev; // File system’s disk device
	uint ino; // Inode number
	short type; // Type of file
	short nlink; // Number of links to file
	uint64 size; // Size of file in bytes
};
```

我们可以使用`fstat`系统调用获取一个文件描述符关联到的`stat`的状态。同时，我们也可以使用`link`系统调用，将一个新的文件名指向已经打开的文件，并且使用这两个文件名可以对同一个文件进行操作：

```C
open("a", O_CREATE|O_WRONLY);
link("a", "b");
```

在上面在代码中，真正文件的`stat->nlink`会被置为2。如果想要解除一个文件名对某个索引节点的链接，可以使用`unlink()`系统调用。

