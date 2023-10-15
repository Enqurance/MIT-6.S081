# MIT 6.S081 Lab1 Util

本系列文档介绍笔者对MIT公开课程6.S081种Lab1 Util各个题目的思路与理解。由于在写作时，笔者的C刚刚重新捡起来，所以代码写的不是很好，请见谅。

## xv6， 启动！

理论上来讲，按照指导书的启动指南操作即可。

笔者在MacOS 14上搭建xv6环境，按照官方文档的操作，执行`brew install riscv-tools`卡顿，完全clone不下来。建议网络情况不佳的小伙伴寻找其他下载源，如网盘等。笔者最后选择参考[知乎文章](https://www.zhihu.com/column/c_1473785171799928832)来进行安装。安装过程不难，make可能会花费一些时间。改文章提供的riscv工具链也可以用在2023年的xv6代码上。

最后，每次在xv6目录下打开新的终端时，需要配置环境变量让make生效

```bash
export RISCV_HOME=/opt/riscv-gnu-toolchain
export PATH=${PATH}:${RISCV_HOME}/bin
source ~/.zshrc
```

代码是如何运作的？当你在xv的代码目录下编写代码，随后执行`make qemu`时，你编写的代码就会变成可执行文件。随后进入QEMU命令行界面，你就可以在该界面下使用这些可执行文件了。

后面将讲解每一个Task。Coding前，建议先阅读实验指导书。

## sleep

> 请参照UNIX的sleep命令，在xv6中实现一个用户级睡眠程序。用户可以指定sleep所持续的时间参数，代码应放在user/sleep.c文件中。

很简单的题目，关键在于帮助我们理解两件事：

- 命令行参数的读取，用户传进来的时间参数怎么获得？
- 系统调用的使用

下面将通过代码和注释来解答相关问题。

```C
/* argc代表参数的数量，argv代表参数的类型 */
/* 如echo hello world，此时argc=3，argv则是"echo","hello","world" */
int main(int argc, char *argv[])
{
    if (argc != 2) { // 命令错误使用的错误处理
        fprintf(2, "Error!Usage: sleep <time>\n");
        exit(1);
    }
    sleep(atoi(argv[1]));	// 调用内核提供的sleep系统调用，来为用户提供服务
    exit(0);	// 退出程序
}
```

`main()`的参数`argc`和`argv`是非常重要的参数，用于接受用户从shell传递的参数。每一个参数都用`char *`指向，以字符串的形式进行保存。

编写完之后，可以退出QEMU界面（Ctrl + A，然后X），并重新`make qemu`，然后再次执行`ls`，就可以发现一个sleep可执行文件了。在QEMU界面输入`sleep [n]`即可使用。

另外，xv6提供了一个离线评测程序`grade-lab-util.py`，在本地终端执行`./grade-lab-util sleep`即可评测你的sleep程序了。

## pingpong

> 编写一个用户级程序，使用 xv6 系统调用在两个进程间通过一对管道"ping-pong"一个字节。父进程应向子进程发送一个字节；子进程应打印"<pid>: received ping"（<pid>是其进程 ID），并将字节写入父进程的管道，然后退出；父进程应从子进程读取字节，打印"<pid>: received pong"，然后退出。请将程序放在 user/pingpong.c文件中。

本题主要考察对管道的理解。管道是一对文件描述符，一个用于读，一个用于写。通常管道打开的方式：

```c
int p[2];
pipe(p);	// p[0]用于读，p[1]用于写
```

管道的更多细节参见文档Chapter1。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, const char *argv[])
{

    if (argc != 1) { //参数错误
        fprintf(1, "Error!Usage: pingpong\n");
        exit(1);
    }

    int p[2];
    pipe(p);
    if(fork() == 0) {
      	// 子进程
        int ret;
        char c = 'X';
        ret = read(p[0], &c, sizeof(char));		// 通过管道的读端读入字符
        if(ret < 0) exit(1);
        fprintf(1, "%d: received ping\n", getpid());	// 读到字符了！按要求输出
        write(p[1], &c, sizeof(char));	// 通过管道的写端写入字符，pong！
        close(p[0]);	// 关闭读写资源
        close(p[1]);
        exit(0);
    } else {
      	// 父进程
        int ret;
        char c = 'X';		// 所要传递的字符
        write(p[1], &c, sizeof(char));	// 通过管道的写端写入字符，ping！
        ret = read(p[0], &c, sizeof(char));
        if(ret < 0) exit(1);
        fprintf(1, "%d: received pong\n", getpid());	// 读到字符了！按要求输出 
        close(p[0]);	// 关闭读写资源
        close(p[1]);
        exit(0);
    }

    exit(0);
}
```

需要阅读Chapter1的内容，理解`pipe()`系统调用的作用和`fork()`的作用，以及`fork()`会对文件描述符带来怎样的影响。总体来说不是很困难。

## primes

> 使用管道和https://swtch.com/~rsc/thread/的设计，为 xv6 编写一个并发的、可以从2-35筛选质数的程序。这个想法要归功于Unix管道的发明者Doug McIlroy。请在user/primes.c文件中编写你的代码。

![img](https://swtch.com/~rsc/thread/sieve.gif)

这个方法从2-35开始筛选，首先确定2是一个质数，将其打印。随后能够确定的是，所有可以被2整除的数字都不是质数了，这样就可以去掉一些数字。

然后我们发现剩余的数字中，最小的是3，那么3也是一个质数，因为比3小的数字中没有除了1以外的3的因数，将3打印。随后能确定的是，所有可以被3整除的数字都不是质数了，这样又可以去掉一批数字。

然后我们发现剩余的数字中，最小的是5，那么5也是一个质数，因为比5小的数字中没有除了1以外的5的因数......如此循环往复，直到达到35，就可以筛选出所有的质数了。

通过不断地递归创建子进程，可以实现上图所示的筛选流程。当然，要注意关闭不需要使用的文件描述符，否则xv6可能会出现资源不足的情况。

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void prime_drop(int cur_int, int end_int, int array[], int len);

int main(int argc, const char *argv[]) {
    int array[35] = {2, 3, 4, 5, 6, 7, 8, 9, 10,
                     11, 12, 13, 14, 15, 16, 17,
                     18, 19, 20, 21, 22, 23, 24, 25,
                     26, 27, 28, 29, 30, 31, 32, 33,
                     34, 35};
    prime_drop(2, 35, array, 33);
    exit(0);
}

void prime_drop(int cur_int, int end_int, int array[], int len) {
    if (cur_int >= end_int) {
        if(cur_int == end_int) fprintf(1, "prime %d\n", cur_int);
        exit(getpid());
    } else {
        int p[2];
        int i, ret_pid;

        pipe(p);
        ret_pid = fork();

        if (ret_pid != 0) {
            close(p[0]);
            fprintf(1, "prime %d\n", cur_int);
            for (i = 0; i <= len; i++) {
                if (array[i] % cur_int != 0) {	// 筛选非质数
                    write(p[1], &array[i], sizeof(int));
                }
            }
            close(p[1]);
            wait(&ret_pid);	// 依据进程符pid，等待子进程结束
            exit(getpid());	// 按照当前进程的pid退出进程
        } else {
            close(p[1]);
            int buf[40];
            int i = 0;
            while (read(p[0], &buf[i], sizeof(int)) && i < end_int) {
                i++;
            }
            close(p[0]);
            prime_drop(buf[0], buf[i - 1], buf, i);
        }
    }
}
```

个人感觉难度还行，不需要很长的时间写完。

### find

> 为xv6编写一个简单版本的UNIX查找程序：查找目录树中具有特定名称的所有文件。请将代码编写在文件user/find.c中

这个程序自己编写可能会有些困难，建议参考ls程序的源代码（位于`user/ls.c`）。

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"

void find_file(char *path, char *filename);

char *fmtname(char *path);

int
main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(2, "Error!Please use the way:find [path] [filename]\n");
        exit(1);
    }
    char *path = argv[1];	// 获取参数，*path是要查找的路径
    char *filename = argv[2];	// 获取参数，*fileame是要查找的文件名
    find_file(path, filename);
    exit(0);
}

// find_file函数采用递归的思想遍历文件目录
void find_file(char *path, char *filename) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, O_RDONLY)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        close(fd);
        return;
    }

    switch (st.type) {
        // stat indicates a DEVICE or a FILE, print info
        case T_DIR:    // type = 1
            // If the path is too long
            if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
                printf("ls: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';
            while (read(fd, &de, sizeof(de)) == sizeof(de)) {
                if (de.inum == 0)
                    continue;
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;

                if (stat(buf, &st) < 0) {
                    printf("ls: cannot stat %s\n", buf);
                    continue;
                }

                if (strcmp(p, filename) == 0) printf("%s\n", buf);
                if (st.type == 1 && strcmp(p, ".") && strcmp(p, "..")) {
                    find_file(buf, filename);
                }
            }
            break;

        case T_DEVICE: // type = 3
        case T_FILE:   // type = 2
            printf("Error!Not a directory");
            break;
    }
    close(fd);
}

// 借用了ls.c中获得文件名称的函数
char *
fmtname(char *path) {
    static char buf[DIRSIZ + 1];
    char *p;

    // Find first character after last slash.
    for (p = path + strlen(path); p >= path && *p != '/'; p--);
    p++;

    // Return blank-padded name.
    if (strlen(p) >= DIRSIZ)
        return p;
    memmove(buf, p, strlen(p));
    memset(buf + strlen(p), ' ', DIRSIZ - strlen(p));
    return buf;
}
```

### xargs

> 为xv6编写一个简单版本的UNIX xargs程序：xargs的参数描述了要运行的命令，该命令从标准输入中读取行，并将该行附加到命令的参数中，然后运行每一行的命令。解决方案应放在user/xargs.c文件中。

当需要将一个命令的输出作为另一个命令的参数时，xargs命令可以派上用场。它从标准输入或管道中读取数据，并将其作为参数传递给另一个命令。

笔者个人认为这个Task才是最难的，当然最主要的原因是笔者对C语言的指针不太熟悉。笔者保留管道传入的参数写的很丑陋，不忍直视，代码就不贴在此处了。可以参考**【PKUFlyingPig】**的代码，笔者认为很有参考价值。

[一个xargs例子](https://github.com/PKUFlyingPig/MIT6.S081-2020fall/tree/util)