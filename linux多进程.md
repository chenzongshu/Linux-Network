
进程(process)是资源分配的最小单位，线程(thread)是处理机调度的最小单位,进程开销大,线程开销小;但是线程core掉会后会引起整个down掉.

每个进程有唯一的PID标识进程。PID是一个从1到32768的正整数，其中1一般是特殊进程init，其它进程从2开始依次编号。当用完32768后，从2重新开始.

下面针对多进程来讲解下

# fork函数

Linux下一个进程在内存里有三部分的数据，就是"代码段"、"堆栈段"和"数据段"

fork函数就是用来创建子进程的.它复制了父进程:

**子进程和父进程使用相同的代码段,子进程复制父进程的堆栈段和数据段**

相当于本来1个进程, 遇到fork()函数后就分叉成两个进程同时执行了.而且这两个进程是互不影响

- 在父进程中，fork返回新创建子进程的进程ID；
- 在子进程中，fork返回0；
- 如果出现错误，fork返回一个负值；

先看一段简单程序：

```
#include "stdio.h"
#include "sys/types.h"
#include "unistd.h"
 
int main()
{
    pid_t pid1;
    pid_t pid2;
 
    pid1 = fork();    #第1步
    pid2 = fork();    #第2步
 
    printf("pid1:%d, pid2:%d\n", pid1, pid2);
}
```

1、请说出执行这个程序后，将一共运行几个进程。
2、如果其中一个进程的输出结果是“pid1:1001,pid2:1002”，写出其他进程的输出结果（不考虑进程执行顺序）。

a、设原来的进程为P0,PID为XXX,执行到第1步fork的时候,分裂为2个进程,P0,P0-1;
b、执行到第2步的时候,注意P0和P-1都要分裂,所以一下子分裂成了4个进程,P0分裂为了P0和P0-2,P0-1分裂为了P0-1和P0-1-1;
c、因为P0在两次分裂中都为父进程，所以(1001,1002)为P0;
d、第一次分裂中,P0为(1001,0),P0-1为(0,0);
e、第二次分裂, P0为(1001,1002),P0-2作为P0子进程,继承1001,为(1001,0);
               P0-1作为父进程,分裂,自身变为(0,1003),子进程P0-1-1为(0,0);
  

>1、总共有4个jinc进程
2、另外3个输出结果为
pid1:1001, pid2:0
pid1:0, pid2:1003
pid1:0, pid2:0

## exit()函数

从上面的介绍中可以知道，可以通过if判断pid是否为0来判断是否为子进程

子进程可以用 exit() 来结束.

## wait()函数

父进程和子进程的执行次序是随机的,  但是实际情况下, 通常我们希望子进程执行后,  才继续执行主进程. 

这个时候可以在父进程中使用`wait()`函数来等待子进程结束

# exec()函数族

需要注意的是exec并不是1个函数, 其实它只是一组函数的统称, 它包括下面6个函数:

```
#include <unistd.h>  
  
int execl(const char *path, const char *arg, ...);  
  
int execlp(const char *file, const char *arg, ...);  
  
int execle(const char *path, const char *arg, ..., char *const envp[]);  
  
int execv(const char *path, char *const argv[]);  
  
int execvp(const char *file, char *const argv[]);  
  
int execve(const char *path, char *const argv[], char *const envp[]);  
```

可以见到这6个函数名字不同, 而且他们用于接受的参数也不同.

exec函数会取代执行它的进程,  也就是说, 一旦exec函数执行成功, 它就不会返回了, 进程结束. 但是如果exec函数执行失败, 它会返回失败的信息,  而且进程继续执行后面的代码!
