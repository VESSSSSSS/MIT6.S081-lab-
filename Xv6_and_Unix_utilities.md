## 1.Sleep

第一个要我们书写的是xv6系统下的sleep函数

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc , char *argv[]) {
    if(argc != 2) {
        printf("an error message , error <ticks>\n");
        exit(1);
    }
    int T = atoi(argv[1]);
    sleep(T);
    exit(0);
}
```

根据官方文档中的提示，我们可以去借鉴其他函数的实现，例如 user/echo.c,user/grep.c 其函数的结构

那xv6系统是如何调用sleep这个函数的呢？也就是其具体底层的实现：

在user/usy.S中 可以找到关于sleep的一个全局定义，然后使用了sleep就将SYS_sleep这个系统调用的代号放入a7这个寄存器中

![img](https://picx.zhimg.com/80/v2-a11bc78f5c81c7b5487473eab4453cd5_720w.png?source=d16d100b)





user/usy.S 中的sleep函数定义

我们再看到  syscall.c  中就可看到一个函数数组，SYS_sleep 和 sys_sleep存在一个映射关系，也就是这样一个关系

```
syscalls[SYS_sleep] == sys_sleep
```

![img](https://picx.zhimg.com/80/v2-2c01eb72f60407e4fc236522d876691d_720w.png?source=d16d100b)





syscall.c

随后在使用sleep函数之后，便通过ecall指令进行一个系统调用去调用我们的sys_sleep函数：

![img](https://pic1.zhimg.com/80/v2-fb9cf6ead936017508179310972e21d4_720w.png?source=d16d100b)





syscall.c

![img](https://picx.zhimg.com/80/v2-85ccff34f54f923064441f8affef8054_720w.png?source=d16d100b)





sysproc.c

然后使我们的当前进程进入一个休眠状态

## 2.pingpong

第二个则是要书写一个可以进行双端传送的管道程序，实验中需要我们实现一个双向的管道，可以在两个进程之间进行互相传送一个字节的数据

![img](https://picx.zhimg.com/80/v2-d411bd35baef331f707c23189fc13562_720w.png?source=d16d100b)





pipe进行进程间通信的图示

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc , char* argv[]) {
    char buf[] = {'w'};
    int parent2child[2];
    int child2parent[2];
    pipe(child2parent);
    pipe(parent2child);
    int pid = fork();
    if (pid == 0) {
        close(parent2child[1]);
        read(parent2child[0], buf, 1);
        printf("%d: received ping\n", getpid());
        write(child2parent[1], buf, 1);
        exit(0);
    }
    else {
        close(child2parent[1]);
        write(parent2child[1], buf, 1);
        read(child2parent[0], buf, 1);
        printf("%d: received pong\n", getpid());
        exit(0);
    }
    exit(0);
}
```

要注意的一点就是，**在在对于一个管道的一端进行read的时候，一定要进行关闭write**，否则read这个操作就会一直等待write操作完成，从而造成一个死锁的状态，这一点在xv6的参考文档中也进行了一个说明

![img](https://pica.zhimg.com/80/v2-09cd3ac691c52a8851713d4dab102109_720w.png?source=d16d100b)





xv6参考文档的译文

所以一定要记得关闭写入端的文件描述符，还有一点就是，在每个进程中，只保留需要使用的文件操作符，也就是需要进行操作的管道端口，其他不需要使用的端口都可以进行关闭

除此之外，还因为子进程和父进程是并发执行的，但我们题目中的输出是需要这样的

![img](https://pic1.zhimg.com/80/v2-036bf934036af6c8f3e4f001a700913f_720w.png?source=d16d100b)





正确的输出

在实验过程中，若没有考虑并发的问题，则会出现这样的输出

![img](https://pica.zhimg.com/80/v2-d5a2fb95a1fc22e706412ea289b65658_720w.png?source=d16d100b)





因为没有考虑并发，从而错误输出

题目要求一定要子进程先输出，然后父进程再输出，所以我们就需要在子进程中卡住对于父进程中的管道中读入端的操作，也就是我们要将printf输出代码放到子进程的write操作前面，在对管道写入端进行操作前，就完成信息的输出，在write端写入未完毕的时候，read端并不会去读取数据，会进行一个等待的状态

**注**：此处对于这样为什么可以很好的处理并发，我也不是很清楚，按照我的思路解释是因为，子进程和父进程共享一根管道，然后对于子进程的写入在printf输出之后，会导致父进程中的read端一直在等待数据的传送，从而达到一个等待子进程执行printf语句的效果，但子进程的写入端是否写入数据为什么会影响父进程的读取端呢？父进程是如何知道子进程会进行一个写入操作呢？我尝试着把子进程的写入操作进行一个注释，任然可以得到一个正确的执行顺序答案（尽管此时已经无法传输一字节数据了），那就说明，父进程中是知道子进程是否进行了写入操作的，并且子进程中是否关闭写入端对父进程的读取无影响

**应该其他进程的写端未关闭是有影响的，找到个pipe的解释：**

![img](https://picx.zhimg.com/80/v2-8a41e0f678b1b496e8c6a25a7ecd1c1d_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

但不知道为什么在pingpong这个实验中，尽管我注释掉了子进程的写端，并且没有去关闭他，但父进程的读入端还是可以正常的读入的，并不会产生阻塞这个效果

![img](https://pica.zhimg.com/80/v2-552f5d9d960797e8b667b45b75279a4c_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

即这样，父进程任然可以正常读入，是因为子进程已经结束了并且其内存已经释放，然后这个写端就对父进程没有影响了嘛（猜的，但其并发执行的话，还是有概率产生阻塞的这样一个状态吧，所以随手关闭不需要的端口很重要）

最后还是带着疑问完成了这个实验（**但这里还是要注意一下，不同进程在公用一个管道进行通信的时候，需要去小心在不同进程间的读入端和写入端的顺序问题否则会造成一些错误**）：

![img](https://pic1.zhimg.com/80/v2-79c4e4920d55a85c327d88ce0d51818b_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

## 3.primes

第三个实验是通过管道并发来实现一个筛法（埃氏筛）

![img](https://picx.zhimg.com/80/v2-80c335f344e08bbc2ef7d682478e1c1f_720w.png?source=d16d100b)





并发的图示

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void next_pros(int* Parent_To_Child) {
    close(Parent_To_Child[1]);
    int n , status;
    if((status = read(Parent_To_Child[0], &n, sizeof(int))) == 0) {
        close(Parent_To_Child[0]);
        exit(0);
    } else if(status == -1) {
        fprintf(2, "error\n");
        exit(1);
    }
    printf("prime %d\n", n);
    int Parent_To_Child_2[2];
    pipe(Parent_To_Child_2);
    int pid = fork(); // 1111
    if(pid == 0) {
        close(Parent_To_Child[0]);
        close(Parent_To_Child_2[1]); // 111
        next_pros(Parent_To_Child_2);
    }
    else
    {
        close(Parent_To_Child_2[0]);

    }
    close(Parent_To_Child[0]);
    close(Parent_To_Child_2[1]); // 1111
    wait((int *)0);
}

int main(int argc , char* argv[]) {
    int Parent_To_Child[2];
    pipe(Parent_To_Child);
    in        int num;
        while (read(Parent_To_Child[0], &num, sizeof(int)) > 0)
        {
            if(num % n != 0)
                write(Parent_To_Child_2[1], &num, sizeof(int));
        }t pid = fork(); //111
    if (pid == 0)
    {
        next_pros(Parent_To_Child);
    }
    else
    {
        for (int i = 2; i <= 35; i++)
        {
            write(Parent_To_Child[1], &i, sizeof(int));
        }
        close(Parent_To_Child[1]); // 1111
    }
    wait((int *)0);
    exit(0);
}
```

这里在实现管道的时候，需要注意死锁的问题，在进行递归的时候，要记得关闭管道写入端，否则就会造成死锁，然后程序无法结束这样的问题，也就是在每次next_pros 之前要记得 close(Parent_To_Child_2[1]); ,剩下就是埃氏筛的实现了，每次筛去当前质数的倍数（大于一倍），复杂度是  

## 4.find

第四个实验需要我们去实现一个在给定地址（目录 / 文件）下，进行查找相应文件的这样一个程序，首先我们可以根据文档提示，去查看ls.c 中ls函数的具体实现方式，然后可以发现主要是运用了 dirent和 stat这两个类型，前者主要是一个索引的作用，其inum为inode 号码用来索引文件文件或目录的stat也就是找到其相对应的具体的文件或目录的相应信息，DIRSIZ是单个文件名的最大长度

![img](https://picx.zhimg.com/80/v2-920c2a72f2ae6855cb55419cd0232383_720w.png?source=d16d100b)





dirent

![img](https://picx.zhimg.com/80/v2-cbc61d59490754f25e04bd376ea65e5e_720w.png?source=d16d100b)





stat

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
char * fmtname(char *path){
    char *p;

    // Find first character after last slash.
    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    return p;
}

void find_file(char *Directory, char *Files) {
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(Directory, 0)) < 0){
    fprintf(2, "find: cannot open %s\n", Directory);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, "find: cannot stat %s\n", Directory);
    close(fd);
    return;
  }
  switch(st.type){
  case T_FILE:
      if (strcmp(fmtname(Directory), Files) == 0) {
            printf("%s \n", Directory);
      }
      break;

  case T_DIR:
    if(strlen(Directory) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("find: path too long\n");
      break;
    }
    strcpy(buf, Directory);
    p = buf+strlen(buf);
    *p++ = '/';
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
        if(de.inum == 0 || strcmp("." , de.name) == 0 || strcmp(".." , de.name) == 0) {
            continue;
        }
        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = '\0';
        if(stat(buf , &st) < 0) {
            printf("find: don't find stat %s\n", buf);
        }
        find_file(buf, Files);
    }
    break;
  }
  close(fd);
}

int main(int argc , char* argv[]) {
    if(argc != 3) {
        exit(1);
    }
    find_file(argv[1] , argv[2]);
    exit(0);
}
```

memmove和memcopy，简单在这提一下这两个函数的区别，这两个函数的作用基本上都是相同的，但在对于拷贝的数据和源数据产生了数据重叠时，那么memmove可以去正确的处理结果，而memcopy则不行，也就是，memcopy是memmove的一个子集

## 5.xargs

第五个实验需要我们去写一个xargs程序，其作用是从标准输入读取参数，并附加到后续的命令中去，按行去读取相关命令，并且xv6文档中已经说明，只需要执行一种参数的情形

![img](https://pic1.zhimg.com/80/v2-4b0f50e72a12657cab1d156155e03671_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

但对于标准输入中的每行数据，xargs的相关参数是依次执行的，每行都会执行

我们的思路就是fork出一个子进程，然后在相应的子进程中进行exec相关程序，最后使用wait去等待当前所有子进程都结束 

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"


int main(int argc , char* argv[]) {
    char *_argv[MAXARG];
    char buf[MAXARG * 100];
    char ch; 
    for (int i = 1; i < argc; i++) {
        _argv[i - 1] = argv[i];
    }
    while(1) {
        int index = 0;
        int cnt = argc - 1; 
        int _begin = 0;
        while(1) {
            if (read(0, &ch, sizeof(ch)) <= 0)
            {
                printf("no std_input!\n");
                exit(0);
            }
            if(ch == ' ' || ch == '\n') {
                buf[index ++ ] = 0;
                _argv[cnt ++ ] = &buf[_begin];
                _begin = index;
                if(ch == '\n') {
                    break;
                }
            } else {
                buf[index ++ ] = ch;
            }
        }
        _argv[cnt] = '\0';
        int pid = fork();
        if(pid == 0) {
            exec(_argv[0], _argv);
        } else {
            wait((int *)0);
        }
    }
    exit(0);
}
```

![img](https://pic1.zhimg.com/80/v2-2c601a8c11a4e8820a80c3174f4c9ce9_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

最后需要补上个time.txt才能拿满分，至此S081的lab1完成。