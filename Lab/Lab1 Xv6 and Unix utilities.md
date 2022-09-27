# LAB1

xv6我运行虚拟环境的是*ubuntu64位20.04.4*,最好不要使用更新的版本，很有可能*make qemu*之后无法运行的情况。

## sleep

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"


int
main(int argc, char const *argv[])
{
    if (argc<2){
        printf("You should print the sleeping time\n");
        exit(1);
    }    

    int time=atoi(argv[1]);

    printf("(nothing happens for a little while )\n");
    sleep(time);

    exit(0);

}
```

运行没有问题，但是我似乎需要学着将错误信息这样输出。

```c
fprintf(2,"usage : sleep <time>\n");
```

## pingpong

这是我写的最初版本，特别需要注意**那就是父子进程中不使用的pipe的文件描述符尽快close掉，特别是写端**。不然数据为空的管道read会堵塞。

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"

int main(){
    int  p1[2];
    int  p2[2];
    pipe(p1);
    pipe(p2);
    int pid;
    char buf1[10];
    char buf2[10];
    if (fork()>0){//主进程
        close(p2[1]);
        close(p1[0]);
        write(p1[1],"p",1);
        close(p1[1]);
        if (read(p2[0],buf1,1)==1){
            close(p2[0]);
            pid=getpid();
            printf("%d: received pong\n",pid);
        }
    }else{
        close(p1[1]);
        close(p2[0]);
        if (read(p1[0],buf2,1)==1){
            close(p1[0]);
            pid=getpid();
            printf("%d: received ping\n",pid);
            write(p2[1],"p",1);
            close(p2[1]);
            exit(0);
        }
    }
    exit(0);
}
```



这是work的，但是我对比了其他人写的代码，缺少对于系统调用出现错误的反馈。

## primes

使用递归的方法，第一次写的时候没有设置递归基，所以陷入了无限递归，一直在创建新的进程，最后直接资源不够卡住了。

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"

#define RD 0
#define WR 1

void prime(int p1[2]){
    close(p1[WR]);
    int p;
    if (read(p1[RD],&p,sizeof(int))!=0){
        fprintf(1,"prime %d\n",p);
    }else{
        exit(0);
    }
    int n;
    int p2[2];
    pipe(p2);
    int pid=fork();
    if (pid>0){
        close(p2[RD]);
        while (read(p1[RD],&n,sizeof(int))!=0)
        {
            if (n%p!=0){
                write(p2[WR],&n,sizeof(int));
            }
        }
        close(p1[RD]);
        close(p2[WR]);
        wait((int *)0);
        exit(0);
    }else if (pid==0){
        prime(p2);
    }
    exit(0);
}


int main(){
    int p1[2];
    pipe(p1);
    int pid=fork();
    if (pid<0){
        fprintf(2,"fork error!\n");
        close(p1[RD]);
        close(p1[WR]);
        exit(1);
    }else if (pid==0){
        prime(p1);
    }else{//main proc
        close(p1[RD]);
        for (int i = 2; i <=35 ; i++)
        {
            write(p1[WR],&i,sizeof(int));     
        }
        close(p1[WR]);
        wait((int *)0);
    }

    exit(0);
    
}
```





## find

（该程序需要查询一个目录树下有特定名称的所有文件）

在测试该程序的时候，需要创建b文件和a/b文件，手册上是这么做的。

```shell
$ echo > b
$ mkdir a 
$ echo > a/b
```

 其实我有点奇怪的是`echo`的代码是这样的，只是做了输出而已。它是怎么创建文件的？

<img src="http://cdn.zhengyanchen.cn/img202206191105442.png" alt="截屏2022-06-19 11.05.15" style="zoom: 40%;" />

稍微思考了，这个操作应该是在`sh.c`做。事实上,`sh.c`也确实这么做了并做了重定向，让文件描述符1重定向到的文件b。

<img src="http://cdn.zhengyanchen.cn/img202206191110226.png" alt="截屏2022-06-19 11.10.44" style="zoom:40%;" />

现在我来重新理解文件描述符：

1. 所有的文件描述符核心都是在服务进程
2. 一个文件可以对应多个文件描述符
3. 可以把文件描述符理解为此进程打开此文件的程度（偏移量）。

回到该lab，要找目录的特定文件，首先阅读`ls.c`。(`.`表示当前目录，`..`表示下上一级目), 然后就可以写了。



```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char* path,char * file){
    int fd;
    struct stat st;
    struct dirent de;
    char buf[512];
    char *p;
    if((fd=open(path,0))<0){
        fprintf(2,"can not open%d\n",path);
        exit(1);
    }
 
    if ((fstat(fd,&st))<0){
        fprintf(2,"can not stat%d\n",path);
        close(fd);
        exit(1);        
    }
    switch (st.type)
    {
    case T_FILE:
        fprintf(2,"please find file in dir");
        close(fd);
        break;
    case T_DIR:
        strcpy(buf, path);
        p = buf+strlen(buf);
        *p++ = '/';
        while(read(fd, &de, sizeof(de)) == sizeof(de)){
            if(de.inum == 0)
            continue;
            memmove(p, de.name, DIRSIZ);
            if(stat(buf, &st) < 0){
                fprintf(2,"find: cannot stat %s\n", buf);
                continue;
            }
            if (st.type==T_DIR){
                if (de.name[0]=='.'){
                    continue;
                }else{
                    find(buf,file);
                }
            }else if(st.type==T_FILE){
                if (strcmp(file,de.name)==0 ){
                    fprintf(1,"%s\n",buf);
                }
            }            
        }
    }

    return ;

}

int main(char argc,char *argv[]){
    if (argc>3){
        fprintf(2,"need less argument\n");
        exit(1);
    }else if (argc==2){
        find(".",argv[1]);
    }else{
        find(argv[1],argv[2]);
    }   
    exit(0); 
}
```





## xargs

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"
#include"kernel/fcntl.h"
#include"kernel/fs.h"
#include"kernel/param.h"
int main(char argc,char * argv[]){
    char buf;
    char command[DIRSIZ];
    char *args[MAXARG]={0};
    int argnum=0;
    if (argc>1){
        strcpy(command,argv[1]);
        while (argnum<(argc-1))
        {
            args[argnum]=argv[argnum+1];
            argnum++;
        }
        char line[128];
        int linenum=0;
        while (read(0,&buf,1)>0)
        {
            if (buf=='\n'){
                line[linenum]=0;
                args[argnum]=line;
                int pid=fork();
                if(pid<0){
                    fprintf(2,"xargs: fork error");
                }else if (pid==0){
                    exec(command,args);
                    exit(0);
                }else{
                    linenum=0;
                    wait((int *)0);
                }
            }else{
                line[linenum++]=buf;
            }
        }
    }else{
        fprintf(2,"a command shoud be given");
        exit(1);
    }
    
    exit(0);
}


```



















