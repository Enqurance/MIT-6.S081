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
exec("/bin/echo", argv);
printf("exec error\n");
```

