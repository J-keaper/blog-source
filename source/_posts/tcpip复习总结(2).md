title: TCP/IP网络编程复习总结(2)
tags:
  - 网络编程
author: Keaper
categories:
  - TCP/IP网络编程复习
date: 2017-06-07 18:47:00
---

# 第十章 多进程服务器端
## 1. 进程该概念及应用
- 并发服务端实现方法
	- 多进程服务器
	- 多路复用服务器
	- 多线程服务器
- 创建进程 fork()函数
	- 头文件：`<unistd.h>`
	- 原型：
    ```c
    pid_t fork(void);
    ```
    
	- 参数：空
	- 返回值：在父进程中fork返回子进程ID，在子进程中返回0
- fork函数示例
    ```c
    #include <stdio.h>
    #include <unistd.h>

    int main()
    {
        pid_t pid;
        int lval =20;

        pid=fork();
        if(pid==0)
            lval+=2;
        else 
            lval-=2;

        if(pid==0)	//子进程
            printf("child proc : %d \n",lval);
        else 		//父进程
            printf("parent proc : %d \n",lval);
    }
    /*
    output:
    parent proc : 18 
    child proc : 22
    */
    ```


## 2. 进程和僵尸进程
- 产生僵尸进程的原因：向exit函数传递的参数值和main函数中的return语句返回的值都会传递给操作系统。但操作系统不会销毁子进程直到把这些值传递给产生该进程的父进程。处于这种状态的进程就是僵尸进程。
- 销毁僵尸进程的方法：向创建子进程的父进程传递子进程的exit参数值和return语句的返回值。

## 3. 销毁僵尸进程

### (1) wait函数
- 头文件：`<sys/wait.h>`
- 功能：等待子进程返回
- 原型：
    ```c
    pid_t wait(int  * statloc);`
    ```
    
- 参数：调用此函数时，如果已有子进程终止，那么子进程终止时传递的返回值将保存到参数所指的内存空间。但参数指向的单元有其他信息，故需要通过宏进行分离。
	- WIFEXITED 子进程正常终止是返回true
	- WEXITSTATUS 返回子进程的返回值
- 返回值：成功返回终止的子进程ID，失败返回-1<br>*调用wait函数时，如果没有已经终止的子进程，那么程序将**阻塞**直到有子进程终止。*
- 示例:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    int status;
    pid_t pid=fork();

    if(pid==0)
        return 3;
    else 
    {
        printf("Child PID : %d \n",pid);
        pid = fork();
        if(pid==0)
            exit(7);
        else 
        {
            printf("Child PID : %d \n",pid);
            wait(&status);
            if(WIFEXITED(status))
                printf("Child send: %d \n",WEXITSTATUS(status));

            wait(&status);
            if(WIFEXITED(status))
                printf("Child send: %d \n",WEXITSTATUS(status));
        }
    }
    return 0;
}
/*
output:
Child PID : 4775 
Child PID : 4776 
Child send: 3 
Child send: 7 
*/
```

### (2) waitpid函数
- 头文件：`<sys/wait.h>`
- 原型：
    ```c
    pid_t waitpid(pid_t pid, int * statloc, inoptions);`
    ```
    
- 参数：
	- pid 等待终止的目标子进程ID，若传递-1，则等待任意子进程终止
	- statloc 与wait函数意义相同
	- options 传递头文件`<sys/wait.h>`中的常量WNOHANG，即时没有终止的子进程也**不会进入阻塞状态**，而是返回0并退出函数。
- 返回值：
	成功是返回终止的子进程ID（或者0），失败时返回-1
- 示例
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main()
{
    int status;
    pid_t pid = fork();

    if(pid==0)
    {
        sleep(15);
        return 24;
    }
    else
    {
        while(!waitpid(-1,&status,WNOHANG))
        {
            sleep(3);
            puts("sleep 3 sec.");
        }

        if(WIFEXITED(status))
            printf("Child send %d \n",WEXITSTATUS(status));
    }
    return 0;
}
/*
output:
sleep 3 sec.
sleep 3 sec.
sleep 3 sec.
sleep 3 sec.
sleep 3 sec.
Child send 24
*/
```

## 4. 信号处理

### (1)信号
- 信号是在特定事件发生时由操作系统向进程发送的信息

### (2)signal函数
- 功能：注册信号
- 头文件：`<signal.h>`
- 原型：
    ```c
    void (*signal(int signo,void(*func)(int)))(int);
    ```
    
- 参数：
	- func : 信号发出时要调用的函数，该函数是参数类型为int，返回void型的函数ji
	- signo: 需要处理的信号
    
| 信号   |    含义 |
|:------:|:------:|
|SIGALRM|已到通过调用alarm函数注册的时间|
|SIGINT|程序终止信号，通常是输入Ctrl+C|
|SIGCHLD|子进程终止|
    

- 返回值：返回之前注册的函数指针（参数类型为int，返回void型的函数）
- 示例:

```c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void timeout(int sig)
{
    puts("time out");
    alarm(2);
}

void keycontrol(int sig)
{
    puts("Ctrl+C pressed");
}
int main()
{
    signal(SIGALRM,timeout);
    signal(SIGINT,keycontrol);
    alarm(2);

    int i;
    for(i=0;i<3;i++)
    {
        puts("wait...");
        sleep(100);
    }
    return 0;
}
/*
output:
wait...
time out
wait...
^CCtrl+C pressed
wait...
time out
*/
```

*注：信号的产生会唤醒由于调用sleep函数而进入阻塞状态的进程，进程一旦被唤醒就不会再进入睡眠状态。*

### (3) alarm函数
- 原型：
```c
unsigned int alarm (unsigned int seconds);`
```

- 参数：如果传递一个正整型参数，则相应时间到达（以秒为单位）后将产生SIGALRM信号。若传递参数0，则取消之前的对 SIGNAL信号的预约。
- 返回值：返回0或者，以秒为单位的距SIGNAL信号发生所剩的时间

### (4) sigaction函数

- 原型：
    ```c
    int sigaction (int signo, const struct sigaction *act,
    				struct sigaction *oldact);
    ```
    
- 参数：
	- signo 传递信号信息
	- act 对应于第一个参数的信号处理函数信息。
	- oldact 通过此参数获取之前注册的信号处理函数指针，若不处理则传递0
- 返回值：成功返回0,失败返回-1
- struct结构体:
    ```c
    struct sigaction
    {
        void (*sa_handler) (void);
        sigset_t sa_mask;
        int sa_flags;
    };
    ```

- 示例：
```c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>

void timeout(int sig)
{
    if(sig==SIGALRM)
        puts("Time out");
    alarm(2);
}

int main()
{
    struct sigaction act;
    //注册信号
    act.sa_handler=timeout;
    sigemptyset(&act.sa_mask);
    act.sa_flags=0;
    sigaction(SIGALRM,&act,0);

    int i;
    alarm(2);
    for(i=0;i<3;i++)
    {
        puts("wait...");
        sleep(100);
    }
    return 0;
}
/*
output:
wait...
Time out
wait...
Time out
wait...
Time out
*/
```

### (5) 利用信号处理技术消灭僵尸进程

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>

void read_child(int sig)
{
    int status;
    pid_t pid = waitpid(-1,&status,WNOHANG);
    if(WIFEXITED(status))
    {
        printf("Remove proc id : %d \n",pid);
        printf("Child send : %d \n",WEXITSTATUS(status));
    }
}

int main()
{
    pid_t pid;
    struct sigaction act;
    act.sa_handler=read_child;
    sigemptyset(&act.sa_mask);
    act.sa_flags=0;
    sigaction(SIGCHLD,&act,0);

    pid=fork();
    if(pid==0)
    {
        sleep(10);
        return 12;
    }
    else 
    {
        printf("Child PID : %d \n",pid);
        int i;
        for(i=0;i<3;i++)
        {
            puts("wait...");
            sleep(5);
        }
    }

    return 0;
}
/*
output:
Child PID : 5119 
wait...
wait...
Remove proc id : 5119 
Child send : 12 
wait...
*/
```

## 5. 基于多任务的并发服务器
- 为每个客户端都创建一个进程提供服务
- 实现并发服务器
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/wait.h>

#define BUF_SIZE 30

void read_child(int sig)
{
    int status;
    pid_t pid = waitpid(-1,&status,WNOHANG);
    printf("Remove proc id ; %d \n",pid);
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}

int main(int argc,char *argv[])
{
    int serv_sock,clnt_sock;
    struct sockaddr_in serv_addr,clnt_addr;
    socklen_t clnt_addr_size;

    char buf[BUF_SIZE];
    pid_t pid;

    if(argc!=2)
    {
        printf("Usage : %s <port>\n",argv[0]);
        exit(1);
    }
    //注册信号
    struct sigaction act;
    act.sa_handler = read_child;
    sigemptyset(&act.sa_mask);
    act.sa_flags= 0;
    if(sigaction(SIGCHLD,&act,0)==-1)
        error_handling("sigaction() error");
    //创建套接字
    serv_sock = socket(PF_INET,SOCK_STREAM,0);
    if(serv_sock==-1)
        error_handling("socket() error");
    //地址初始化
    memset(&serv_addr,0,sizeof(serv_addr));
    serv_addr.sin_family=AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));

    if(bind(serv_sock,(struct sockaddr*) &serv_addr,sizeof(serv_addr))==-1)
        error_handling("bind() error");

    if(listen(serv_sock,5)==-1)
        error_handling("listen() error");

    while(1)
    {
        clnt_addr_size = sizeof(clnt_addr);
        clnt_sock = accept(serv_sock,(struct sockaddr*) &clnt_addr,&clnt_addr_size);
        if(clnt_sock==-1)
            continue;
        else 
            puts("new connection !");

        pid = fork();
        if(pid==0) //子进程
        {
            close(serv_sock); //子进程关闭监听套接字
            int str_len;
            while((str_len=read(clnt_sock,buf,BUF_SIZE))!=0)
                write(clnt_sock,buf,str_len);

            close(clnt_sock);
            puts("client disconnected !");
            return 0;
        }
        else
        {
            close(clnt_sock);  //父进程关闭数据传输的套接字
        }
    }
    close(serv_sock);
    return 0;
}
```

## 6. 分割TCP的I/O程序


# 第十一章 进程间通信

## 1. 进程件间通信的基本概念

**创建管道 pipe() 函数**

- 原型：
```c
int pipe(int filedes[2]);`
```

- 参数：
	- filededs[0] 通过管道接受数据使用的文件描述符，即管道出口
	- filededs[1] 通过管道发送数据使用的文件描述符，即管道入口
- 返回值：成功返回0,失败返回-1

**通过管道实现进程间通信**
```c
#include <stdio.h>
#include <unistd.h>
int main()
{
	int fds[2];
	pipe(fds);
	char str[]="Who are you?";
	char buf[30];
	
	pid_t pid = fork();
	if(pid==0)
	{
		write(fds[1],str,sizeof(str));
	}
	else
	{
		read(fds[0],buf,sizeof(buf));
		puts(buf);
	}
	return 0;
}

/*
output:
Who are you?
*/
```
**通过管道进行进程间双向通信**

一个管道在需要双向通信时会产生混乱，所以双向通信时需要两个管道用于不同方向的数据传输。
```c
#include <stdio.h>
#include <unistd.h>

int main()
{
	int fds1[2],fds2[2];
	pipe(fds1);
	pipe(fds2);
	char str1[]="How are you?";
	char str2[]="I'm fine.";
	char buf[30];
	
	pid_t pid =fork();
	
	if(pid==0)
	{
		write(fds1[1],str1,sizeof(str1));
		read(fds2[0],buf,30);
		printf("Child proc print: %s \n",buf);
	}
	else
	{
		read(fds1[0],buf,30);
		printf("Parent proc print: %s \n",buf);
		write(fds2[1],str2,sizeof(str2));
		sleep(2);
	}
	return 0;
}
/*
output:
Parent proc print: How are you? 
Child proc print: I'm fine.
*/
```

# 第十二章 I/O复用
多进程服务器端中，创建进程的代价较大，进程间相互通信也相对复杂。
## 1. select函数
- 使用selct函数可以将多个文件描述符集中到一起统一监视。
- slect函数的调用顺序：<br>
	**设置文件描述符，指定监视范围，设置超时 ===> 指定监视范围 ===> 调用select函数 ===> 查看调用结果**
    
**设置文件描述符**

- select函数可以同时监视多个文件描述符，可以视为监视多个套接字。
- 首先要将监视的文件描述符集中在一起，集中时按照监视项（接受，传输，异常）进行区分。
- fd_set数组变量，每一位状态都表示文件描述符为此下标的文件或者套接字是否处于监视范围，1表示是监视对象，0表示不是。
- 操作fd_set的宏

|   宏   |   操作 |
|:------:|:------:|
|FD_ZERO(fd_set *fdset) |将fd_set变量的所有位初始化为0|
|FD_SET(int fd,fd_set *fdset) |在参数fdset指向的变量中注册文件描述符fd的信息，即将该位置为1|
|FD_CLR(int fd,fd_set *fdset) |从参数fdset指向的变量中清除文件描述符fd的信息，即将该位置为0|
|FD_ISSET(int fd,fd_set *fdset)|若参数fdset指向的变量中包含文件描述符fd的信息，则返回true |

**设置监视范围及超时select函数**

- 头文件`<sys/select.h>`
- 原型：
```c
int select (int maxfd, fd_set *readfds,
       fd_set *writefds, fd_set *exceptfds,
       const struct timeval *timeout);
```
- 参数：
	- maxfd 监视对象文件描述符的数量
	- readfds 将所有关注"是否存在待读取数据"的文件描述符注册到fd_set型变量，并传递其地址值
	- writefds 将所有关注"是否可传输无阻塞数据"的文件描述符注册到fd_set型变量，并传递其地址值
	- exceptfds 将所有关注"是否发生异常"的文件描述符注册到fd_set型变量，并传递其地址值
	- timeout 设置超时时间
- timeval结构体
    ```c
    struct timeval
    {
        long sec;	“”	/* Seconds.  */
        long usec;	/* Microseconds.  */
    };
    ```
    
- 返回值：发生错误返回-1,超时返回0。发生关注的事件返回时，返回大于0的值，该值是发生事件的文件描述符数量。

**调用sselect函数后查看返回结果**

select函数调用完成后，向其传递的fd_set变量中将发生变化。原来为1的所有位均变为0,但发生变化的文件描述符除外。也就是值为1的位置上的文件描述副发生了变化。

**select函数调用示例**
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/select.h>

int main()
{
	fd_set reads,temps;
	struct timeval timeout;
	char buf[30];
	
	FD_ZERO(&reads);
	FD_SET(0,&reads);
	
	while(1)
	{
		temps = reads;
		timeout.tv_sec = 5;
		timeout.tv_usec = 0;
		int result = select(1,&temps,0,0,&timeout);
		
		if(result==-1)
		{
			puts("select() error!");
			break;
		}
		else if(result==0)
		{
			puts("timeout!");
		}
		else 
		{
			if(FD_ISSET(0,&temps))
			{
				int strlen=read(0,buf,sizeof(buf));
				buf[strlen]=0;
				printf("message from console: %s",buf);
			}
		}
	}
	return 0;

}

/*
output:
hello
message from console: hello
hello
message from console: hello
world
message from console: world
timeout!
timeout!
timeout!
timeout!
*/
```

**实现I/O复用的服务器端**
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <sys/select.h>

#define BUF_SIZE 100

void error_handling(char *buf)
{
	fputs(buf, stderr);
	fputc('\n', stderr);
	exit(1);
}


int main(int argc,char *argv[])
{
	int serv_sock,clnt_sock;
	struct sockaddr_in serv_addr,clnt_addr;
	socklen_t clnt_addr_size;
	//监视范围fd_set变量
	fd_set reads,cp_reads;
	struct timeval timeout;
	
	char buf[BUF_SIZE];
	
	if(argc!=2)
	{
		printf("Usage : %s <port>\n",argv[0]);
		exit(1);
	}
	//创建套接字变量
	serv_sock = socket(PF_INET,SOCK_STREAM,0);
	if(serv_sock==-1)
		error_handling("socket() error");
	//地址信息初始化
	memset(&serv_addr,0,sizeof(serv_addr));
	serv_addr.sin_family=AF_INET;
	serv_addr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_addr.sin_port = htons(atoi(argv[1]));
	
	if(bind(serv_sock,(struct sockaddr*) &serv_addr,sizeof(serv_addr))==-1)
		error_handling("bind() error");
	
	if(listen(serv_sock,5)==-1)
		error_handling("listen() error");
	
	FD_ZERO(&reads);
	FD_SET(serv_sock,&reads);	//监视服务端套接字
	int fd_max=serv_sock+1;		//设置最大监视数量

	while(1)
	{
		cp_reads = reads;		//备份fd_set,因为调用select函数后，fd_set会被清空
		timeout.tv_sec = 5;
		timeout.tv_usec = 5000;
		int fd_num;
		if((fd_num=select(fd_max+1,&cp_reads,0,0,&timeout))==-1)	//调用select函数
			break;
		if(fd_num==0)	//超时
			continue;
		int i;
		for(i=0;i<fd_max+1;i++)		
		{
			if(FD_ISSET(i,&cp_reads))
			{
				if(i==serv_sock) //服务端监听套接字发生变化说明，有新的连接请求
				{
					clnt_sock = accept(serv_sock,(struct sockaddr*) &clnt_addr,&clnt_addr_size);
					FD_SET(clnt_sock,&reads);	//将新连接置于监控范围内
					if(fd_max<clnt_sock)
						fd_max=clnt_sock;		//更新最大监控数量
					printf("Connected client : %d \n",clnt_sock);
				}
				else 			//用于数据传输的套接字发生变化，说明有数据需要读取
				{
					int str_len=read(i,buf,sizeof(buf));
					if(str_len==0)		//客户端关闭连接
					{		
						FD_CLR(i,&reads);		//移出监控范围
						close(i);			
						printf("Closed clinet : %d \n",i);
					}
					else 
						write(i,buf,str_len);
				}
			}
		}
		
	}
	close(serv_sock);
	return 0;
}
```

# 第十三章 多种I/O函数
## 1. send() & recv()函数

**send()**

+ 头文件：`<sys/socket.h>`
+ 原型：
```c
ssize_t send (int sockfd, const void *buf, size_t nbytes, int flags);
```
+ 参数：
	- sockfd 表示与发送数据连接的套接字文件描述符
	- buf 待发送数据的缓冲地址值
	- nbytes 待传输的字节数
	- flags 可选项
+ 返回值：成功返回发送的字节数，失败返回-1

**recv()**

+ 头文件：`<sys/socket.h>`
+ 原型：
```c
ssize_t recv (int sockfd, void *buf, size_t nbytes, int flags);
```
+ 参数：	
	- sockfd 表示与接受数据连接的套接字文件描述符
	- buf 保存接受数据的缓冲区地址值
	- nbytes 可接收的最大字节数
	- flags可选项
+ 返回值 ：成功返回收到的字节数（收到EOF返回0），失败返回-1

**flags选项**

|flags	|说明	|recv	|send|
|:------:|:------:|:------:|:------:|
| MSG_OOB　　　　	|发送或接收带外数据(Out-of-band data)	|•		|•		|
| MSG_PEEK　　	|验证输入缓冲是否存在待接受的数据		  	|•		|		|
| MSG_DONTROUTE	|绕过路由表查找，在本地网络中寻找目的地 	| 	 	|•		|
| MSG_DONTWAIT	|调用I/O函数时不阻塞，用于使用非阻塞I/O	|•    	|•		|
| MSG_WAITALL　　	|防止函数返回，直到接受全部请求的字节数	|•		|	 	|

**MSG_OOB选项：发送紧急消息**

- MSG_OOB可选项用于发送“带外数据”紧急消息。
- 示例:

发送端：
```c
... 		//头文件
int main(int argc, char *argv[])
{
	int sock;
	struct sockaddr_in recv_adr;
	if(argc!=3) {
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	sock=socket(PF_INET, SOCK_STREAM, 0);
	memset(&recv_adr, 0, sizeof(recv_adr));
	recv_adr.sin_family=AF_INET;
	recv_adr.sin_addr.s_addr=inet_addr(argv[1]);
	recv_adr.sin_port=htons(atoi(argv[2]));
	
	if(connect(sock, (struct sockaddr*)&recv_adr, sizeof(recv_adr))==-1)
		error_handling("connect() error!");
	
	write(sock,"123",strlen("123"));
	send(sock,"4",strlen("4"),MSG_OOB);  //发送带外数据
	write(sock,"567",strlen("567"));
	send(sock,"890",strlen("890"),MSG_OOB);		//发送带外数据
	
	close(sock);
	return 0;
}
```
接受端：

```c
...			//头文件省略
int acpt_sock;
int recv_sock;

void urg_handler(int sig)           //带外数据处理函数
{
	int str_len;
	char buf[BUF_SIZE];
	str_len=recv(recv_sock,buf,sizeof(buf),MSG_OOB);
	buf[str_len]=0;
	printf("Uegent message : %s \n",buf);
}
int main(int argc, char *argv[])
{
	struct sockaddr_in recv_adr, serv_adr;
	int str_len, state;
	socklen_t serv_adr_sz;
	struct sigaction act;
	char buf[BUF_SIZE];
	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]); 
		exit(1);	
	}
	
	act.sa_handler = urg_handler;
	sigemptyset(&act.sa_mask);
	act.sa_flags=0;
	
	acpt_sock = socket(PF_INET,SOCK_STREAM,0);

	memset(&recv_adr, 0, sizeof(recv_adr));
	recv_adr.sin_family=AF_INET;
	recv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	recv_adr.sin_port=htons(atoi(argv[1]));

	if(bind(acpt_sock, (struct sockaddr*)&recv_adr, sizeof(recv_adr))==-1)
		error_handling("bind() error");
	listen(acpt_sock, 5);
	serv_adr_sz=sizeof(serv_adr);
	recv_sock=accept(acpt_sock, (struct sockaddr*)&serv_adr, &serv_adr_sz);
	
	fcntl(recv_sock,F_SETOWN,getpid());     //为带外数据指定处理进程
	state=sigaction(SIGURG,&act,0);         //为SIGURG信号注册函数

	while((str_len=recv(recv_sock,buf,sizeof(buf),0))!=0)   //正常读取普通数据
	{
		buf[str_len]=0;
		puts(buf);
	}
	
	close(recv_sock);
	close(acpt_sock);
	
	return 0;
}
```
**检查输入缓冲**

- 设置MSG_PEEK选项并调用recv函数时，即使读取了输入缓冲中的数据也不会删除。因此该选项经常与MSG_DONTWAIT合作，用于调用以非阻塞方式验证待读数据的存在与否的函数。
- 示例:

```c
... 			//头文件省略
void error_handling(char *message);
int main(int argc, char *argv[])
{
	int acpt_sock, recv_sock;
	struct sockaddr_in acpt_adr, recv_adr;
	int str_len, state;
	socklen_t recv_adr_sz;
	char buf[BUF_SIZE];
	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	
	acpt_sock=socket(PF_INET, SOCK_STREAM, 0);
	memset(&acpt_adr, 0, sizeof(acpt_adr));
	acpt_adr.sin_family=AF_INET;
	acpt_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	acpt_adr.sin_port=htons(atoi(argv[1]));
	
  	if(bind(acpt_sock, (struct sockaddr*)&acpt_adr, sizeof(acpt_adr))==-1)
		error_handling("bind() error");
	listen(acpt_sock, 5);
	
	recv_adr_sz=sizeof(recv_adr);
	recv_sock=accept(acpt_sock, (struct sockaddr*)&recv_adr, &recv_adr_sz);
	
	while(1)
	{
		//查看缓冲是否有数据，读完 不会删除缓冲区
		str_len=recv(recv_sock,buf,sizeof(buf)-1,MSG_PEEK|MSG_DONTWAIT); 		
		if(str_len>0)
			break;
	}
	buf[str_len]=0;
	printf("Buffering %d bytes: %s \n", str_len, buf);
	
	//正常读取，读完删除
	str_len = recv(recv_sock,buf,sizeof(buf)-1,0);
	buf[str_len]=0;
	printf("Readagin : %s \n",buf);

	close(acpt_sock);
	close(recv_sock);
	return 0; 
}
```
## 2. readv & writev函数
通过writev函数可以将分散保存在多个缓冲中的数据一并发送，通过readv函数可以由多个缓冲分别接受。

**writev函数**

- 头文件：`<sys/uio.h>`
- 原型：
    ```c
    ssize_t writev (int fd, const struct iovec *iov, int iovcnt);
    ```
- 参数：
	- fd 表示数据传输对象的文件描述符
	- iov iov结构体数组的地址值
	- iovcnt 第二个参数的数组长度
- 返回值：成功时返回 发送的字节数，失败返回-1
- iovec结构体
    ```c
    struct iovec
    {
        void *iov_base;	/* Pointer to data.  */
        size_t iov_len;	/* Length of data.  */
    };
    ```
- 示例:

```c
#include <stdio.h>
#include <sys/uio.h>

int main()
{
	struct iovec vec[2];
	char buf1[]="ABCDEFG";
	char buf2[]="1234567";
	vec[0].iov_base = buf1;
	vec[0].iov_len=3;
	vec[1].iov_base = buf2;
	vec[1].iov_len=4;
	
	int str_len=writev(1,vec,2);
	printf("\nWrite bytes: %d \n",str_len);	

	return 0;
}
```

**readv函数**

- 头文件：`<sys/uio.h>`
- 原型：
```c
ssize_t readv (int fd, const struct iovec *iov, int iovcnt);
```
- 参数：
	- fd 表示数据接收对象的文件描述符（或套接字）
	- iov iov结构体数组的地址值
	- iovcnt 第二个参数的数组长度
- 示例:

```c
#include <stdio.h>
#include <sys/uio.h>
#define BUF_SIZE 100

int main()
{
	struct iovec vec[2];
	char buf1[BUF_SIZE]={0};
	char buf2[BUF_SIZE]={0};
	
	vec[0].iov_base = buf1;
	vec[0].iov_len=5;
	vec[1].iov_base = buf2;
	vec[1].iov_len=20;
	
	int str_len = readv(0,vec,2);
	printf("Read bytes: %d \n",str_len);
	printf("First message: %s \n", buf1);
	printf("Second message: %s ", buf2);
	return 0;
}
```

# 第十四章 多播与广播
## 1. 多播
- 多播方式的数据传输是基于UDP完成的。采用多播方式时，可以同时向多个主机传输数据。
- 多播组是D类IP地址,第1个字节的前四位固定为1110 , 范围：224.0.0.0~239.255.255.255
- 多播是借助路由器复制数据包完成的
- TTL设置方法：通过设置套接字选项

```c
int recv_sock;
struct time_live = 64;
…………
send_sock = socket(PF_INET,SOCK_DGRAM,0);
setopt(send_sock,IPPROTO_IP,IP_MULTICAST_TTL,(void*)&time_live,sizeof(time_live));
…………
```
- 加入多播组的方法：通过设置套接字选项

```c
int recv_sock;
struct ip_mreq join_adr;
…………
recv_sock = socket(PF_INET,SOCK_DGRAM,0);
…………
join_adr.imr_multiaddr.s_addr="<多播组地址信息>";
join_adr.imr_interface.s_addr="<加入多播组的主机地址信息>";
setsockopt(recv_sock,IPPROTO_IP,IP_ADD_MEMBERSHIP,(void*) &join_adr,sizeof(join_adr));
…………
```


## 2. 广播
- 广播也是基于UDP完成的
- 广播分为两种：直接广播和本地广播。
- 直接广播的IP地址中除了网络地址外，其余主机地址全部为1。可以采用直接广播的方式向特定区域内的所有字节传输数据。
- 本地广播的IP地址限定为255.255.255.255。
- 设置允许进行数据广播
```c
int send_sock;
int bast = 1;
…………
send_sock = socket(PF_INET,SOCK_STREAM,0);
…………
setsockopt(send_sock,SOL_SOCKET,SO_BROADCAST,(void *)&bast,sizeof(bast));
```

# 第二十四章 制作HTTP服务器端
- HTTP，Hypertext Transfer Protocol, 超文本传输协议，基于TCP/IP实现的以超文本传输为目的的应用层协议。
- HTTP协议是无状态的协议，服务端响应客户端后立即断开连接
- 状态码：表示客户端请求的执行结果
	- 200 OK：成功处理了请求
	- 404 Not Found：请求的文件不存在
	- 400 Bad Request：错误的请求