#Lab1

##sleep
###代码(抄的)
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(2, "Usage: %s <num> \n", argv[0]);
        exit(1);
    }
    if (sleep(atoi(argv[1])) < 0)
    {
        fprintf(2, "Sleep error\n");
    }
    exit(0);
}

```
###知识点：
1.shell命令程序，比如echo，它们的头文件的目录是相对于shell程序工作目录的，因为是通过shell调用的比如shell在xv6目录下，echo.c在xv6/user目录下，echo.c的一个头文件的引入是这样的#include "user/user.h"，显然，这个路径是相对于shell程序工作目录的，所以echo.c也不能直接在自己的目录下编译,文中提到的信息只符合vx6系统，在别的系统应该有细节上的不同的

2.linux的一个tick 40毫秒，是两次中断的间隔时间，可以看成心跳,sleep参数是tick数量，C库函数sleep的实现是通过在一定时间挂起当前进程，时间结束将进程转为就绪态实现的

##pingpong
###代码
```c
#include "kernel/types.h"
#include "user/user.h"
#include "stddef.h"
int main(){
    int fd[2];
    pipe(fd);
    char rec[2];
    int pfd=fork();
    if(pfd==0){
        read(fd[0],rec,1);
        printf("%d: received ping\n",getpid());
        write(fd[1],"1",1);
    }else{
        write(fd[1],"1",1);
        wait(NULL);
        read(fd[0],rec,1);
        printf("%d: received pong\n",getpid());
    }
    exit(0);
}
```
###知识点
1.正常来说，父子进程双向通信是需要两个管道的，但我通过wait从而使用一个管道进行双向通信

2.received不要写成recived。。。。

3.wait参数是子进程终止时的状态码（通常表示终止原因，是个int指针），返回值是子进程号，这里都没用上

##primes
###代码
```c
#include "kernel/types.h"
#include "user/user.h"
#include "stddef.h"
#define max(a,b) ((a)>(b)?(a):(b))
int maxfd;
void run(int readfd){
    int first,x;
    int n=read(readfd,&first,sizeof(first));
    if(n==0){
        exit(0);
        close(readfd);
    }
    int rfd[2];
    pipe(rfd);
    maxfd=max(maxfd,max(rfd[0],rfd[1]));
    if(fork()){
        close(rfd[0]);//子进程管道读端
        printf("prime %d\n",first);
        while(1){
            n=read(readfd,&x,sizeof(x));
            if(n==0)break;
            if(x%first!=0){
                write(rfd[1],&x,sizeof(x));
            }
        }
        close(readfd);
        close(rfd[1]);
        wait(NULL);
    }else{
        close(rfd[1]);//父进程管道写端
        close(readfd);
        run(rfd[0]);
    }
}

int main(){
    int fd[2];
    pipe(fd);
    if(fork()){
        close(fd[0]);
        int first=2;
        printf("prime %d\n",first);
        for(int i=2;i<=35;i++){
            if(i%first!=0){
                write(fd[1],&i,sizeof(i));
            }
        }
        close(fd[1]);
        wait(NULL);
    }else{
        close(fd[1]);
        run(fd[0]);
    }
    //printf("maxfd=%d\n",maxfd);
    exit(0);
}
```
###注意点or知识点
1.下图表明这是一个埃氏筛，不过是在管道上实现的，也就是说可以通过多个处理器实现线性复杂度的素数筛（可能有更好的解释）
![image](images/pipe_sieve.jpg)

2.对于一个进程，与父进程的管道的写端是无用的，与子进程的管道的读端也是无用的，所以可以直接关闭以确保进程使用的文件描述符数量不超过vx6系统限制（很少，不关闭的话不够用），实际上，任何一个进程会真正用到的文件描述符只有两个，同时存在的文件描述符最多3个（不包括012）

3.管道写端关闭时（写端文件描述符引用为0），读端read不会立即返回0，而是会读取完缓存区数据之后再调用read才返回0

##find
###代码
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char*path,char*name){
    struct dirent de;
    struct stat st,st1;
    char buf[512],*p;
    int fd=open(path,0);
    if(fd<0){
        fprintf(2,"open path %s failed\n",path);
        return;
    }
    if(fstat(fd,&st)){
        fprintf(2,"get access to path %s state failed\n",path);
        close(fd);
        return;
    }
    if(st.type==T_FILE){
        fprintf(2,"path %s should be a dirtory\n",path);
        return;
    }
    strcpy(buf,path);
    p=buf+strlen(buf);
    *p='/';
    ++p;
    while(read(fd,&de,sizeof(de))){
        if(de.inum==0)continue;
        memmove(p,&de.name,DIRSIZ);//name的长度未必有DIRSIZ，这里是为了对齐，但是ls需要对齐，find应该不用，这里可以用strcpy替换
        p[DIRSIZ]=0;
        int fd1=open(buf,0);
        fstat(fd1,&st1);
        if(st1.type==T_FILE){
            if(strcmp(name,de.name)==0){
                printf("%s\n",buf);
            }
        }else if(st1.type==T_DIR){
            if(strcmp(de.name,".")!=0&&strcmp(de.name,"..")!=0)find(buf,name);
        }
        close(fd1);
    }
    close(fd);
}

int main(int argc,char *argv[]){
    if(argc<3){
        fprintf(2,"find <dir> <name>...\n");
        exit(1);
    }
    for(int i=2;i<argc;i++){
        find(argv[1],argv[i]);
    }
    exit(0);
}
```
###注意点：
1.读取目录时第一个读取到的文件结构体de，其中de.inum是0，而de.name的值是空字符，记得跳过

2.基本上参考ls.c这个文件就可以搞定这个实验，不过ls因为输出需要对齐，而find不用，所以memmove函数可以替换成strcpy

3.总的思路是读取提供的目录，遇到文件就判断是否匹配，遇到目录就递归判断

##xargs
###代码
```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"
char *argv1[MAXARG+1];
int main(int argc,char *argv[]){
    if(argc<2){
        fprintf(2,"xargs <argv>\n");
        exit(1);
    }
    int argc1=0;
    for(int i=1;i<argc;i++){
        argv1[argc1++]=argv[i];
    }
    char buf[512]={},last[1024]={};
    int size=0;
    while(1){
        int n=read(0,buf,sizeof(buf));
        if(n==0)break;
        for(int j=0;j<n;j++){
            if(buf[j]==' '){
                last[size]=0;
                argv1[argc1]=(char*)malloc(strlen(last)+1);
                strcpy(argv1[argc1++],last);
                size=0;
            }else if(buf[j]=='\n'){
                if(size!=0){
                    last[size]=0;
                    argv1[argc1]=(char*)malloc(strlen(last)+1);
                    strcpy(argv1[argc1++],last);
                    size=0;
                }
                argv1[argc1]=0;
                if(fork()==0){
                    exec(argv[1],argv1);
                }   
                else{
                    argc1=argc-1;
                }         
            }else{
                last[size++]=buf[j];
            }
        }
    }
    if(size!=0){
        last[size]=0;
        argv1[argc1]=(char*)malloc(strlen(last)+1);
        strcpy(argv1[argc1++],last);
    }
    argv1[argc1]=0;
    if(argc1>argc-1){
        exec(argv[1],argv1);
    }
    exit(0);
}
```
注意点：
1.kernel/param.h头文件中设置了各种参数的范围，MAXARG是最大参数数量，根据这个定义exec指针数组参数char *argv1\[MAXARG+1](最后一个元素置零)

2.要求是对每行输入都作为一次命令的参数，所以要多次调用exec，所以要用fork让子进程去调用exec，此外还要根据空格符分隔输入作为参数

3.管道会将管道右命令的输出作为标准输入传给管道左命令，所以xargs直接从标准输入获取输入即可
