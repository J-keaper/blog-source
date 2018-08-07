title: TCP/IP网络编程复习总结(1)
tags:
  - 网络编程
categories: []
author: Keaper
date: 2017-06-06 13:23:00
---
# 第一章 理解网络编程和套接字
## 1. 理解网络编程和套接字
- 网络编程就是编写程序使得两台计算机相互交换数据。
- 套接字是网络传输所用的软件设备。

## 2. 基于linux的文件操作
- 对于linux而言，socket操作和文件操作没有区别。socket也被认为是文件的一种，因此在网络数据传输过程中可以使用文件I/O的相关函数。而Windows是区分socket与文件的。
- 文件描述符：系统分配给文件或套接字的一个整数，用以标识文件或者套接字。	
- 标准输出输出以及标准错误文件描述符：

|       对象  |  文件描述符|
| :--------:  | :-----: |
|标准输入：stdin      |   0   |
|标准输出：stdout      |   1   |
|标准错误：stderr      |   2   |

## 3. 文件操作
**打开文件open()**

* 头文件：`<sys/types.h>`,`<sys/stat.h>`, `<fcntl.h>`<br>*注：文件的打开模式定义在头文件`<sys/stat.h>`中，`<sys/types.h>`中定义了各种数据类型，open()函数定义在`<fcntl.h>`头文件中。*
* 函数原型：
```c
int open(const str * path, int flag);
```
* 返回值：成功则返回文件描述符，出错返回-1 
* 参数：
  * path: 打开或创建的文件的路径名 
  * flag：文件打开模式信息
  
|打开模式|含义|
|:-----:|:------:|
|O_CREAT|必要时创建文件|
|O_TRUNC|删除全部现有数据|
|O_APPEND|维持现有数据，白村到其后面|
|O_RDONLY|只读打开|
|O_WRONLY|只写打开|
|O_RDWR|读写打开|

**关闭文件close()**

* 需要包含的头文件：`<unistd.h>`
* 函数原型：
```c
int close(int fd);
```
* 参数：
	* fd:需要关闭文件的文件描述符

**读取文件中的数据read()**
* 需要包含的头文件：`<unistd.h>`
* 函数原型：
	```c
    ssize_t read(int fd, void * buf, size_t nbytes);
	```
* 返回值：成功时返回实际读到的字节数；已读到文件尾返回0，出错的话返回-1。
* 参数：
	* fd：要读取的文件的描述符
  	* buf：得到的数据在内存中的位置的首地址
  	* nbytes：要接受数据的最大字节数

**向文件中写数据write()**

* 需要包含的头文件：`<unistd.h>`
* 函数原型：
```c
ssize_t write(int fd, const void * buf, size_t nbytes);
```
* 功能：向打开的文件写数据
* 返回值：写入成功返回实际写入的字节数，出错返回-1
* 参数：
  * fd：要写入文件的文件描述符
  * buf：要写入文件的数据在内存中存放位置的首地址
  * nbytes：要传输数据的最大字节数
* 样例代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main()
{
	int fd;
	char buf[]="Hello World!";
	
	fd=open("data.txt",O_CREAT|O_WRONLY|O_TRUNC); 	//打开文件
	printf("File descriptor : %d \n",fd);

	write(fd,buf,sizeof(buf));	//写数据
	close(fd);					//关闭文件
	return 0;
}
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main()
{
	int fd;
	char buf[100];
	
	fd=open("data.txt",O_RDONLY);		//打开文件
	printf("File descriptor: %d \n" , fd);
	
	read(fd,buf,sizeof(buf));			//读数据
	
	printf("File data: %s\n",buf);

	close(fd);			//关闭文件
	return 0;
}
```

# 第二章 套接字类型与协议设置
## 1. 创建套接字socket()
* 需要包含的头文件 <sys/socket.h>
* 函数原型：
```c
int socket(int domain, int type, int protocol);
```
* 返回值：成功：文件描述符；否则：-1。
* 参数：
    * domain：套接字所使用的协议族
    * type：套接字数据传输类型
    * protocol：套接字所使用的协议

## 2. 参数domain:常用协议族
|名称|协议族|
| :-----: | :-----: |
|PF_INET 	|IPv4互联网协议族|
|PF_INET6 	|IPv46互联网协议族|
|PF_LOCAL|本地通信的UNIX协议族|

## 3. 参数type:套接字类型
- 类型1：面向连接的套接字（SOCK_STREAM）<br>**可靠、按序传递的、基于字节的面向连接的套接字。**

  * 传输过程中数据不会消失：错误重传
  * 按序传输数据：按发送顺序放入buffer
  * 传输的数据不存在数据边界
  * 面向连接的套接字只能与另外一个同样特性的套接字连接，一一对应
  
- 类型2：面向消息的套接字（SOCK_DGRAM）<br>**不可靠的、不按序传递、以数据的高速传输为目的的套接字。**

    * 快速传输而非传输顺序
    * 传输的数据可能丢失也可能损毁
    * 传输的数据有边界
    * 每次传输的数据大小有限

## 4. 参数protocol:协议类型
- 如果一个协议族中存在多种数据传输方式，该参数用以确定最终采用的协议。如果前两个参数已唯一确定，这个参数传0即可。

- 创建TCP套接字：
```c
int tcp_socket = socket(PF_INET,SOCKET_STREAM,0);
int tcp_socket = socket(PF_INET,SOCKET_STREAM,IPPROTO_TCP);
```
- 创建UDP套接字：
```c
int tcp_socket = socket(PF_INET,SOCKET_DGRAM,0);
int tcp_socket = socket(PF_INET,SOCKET_DGRAM,IPPROTO_UDP);
```

# 第三章 地址族与数据序列
## 1. IP地址与端口号
- IPv4的地址有4字节，32位。
- 端口号有16位构成，范围为0～65535。
- 端口号就是为了在同一个操作系统内区分不同套接字而设置的，无法将一个端口号分配给不同的套接字。

## 2. 地址信息的表示
```c
struct sockaddr_in
{
    sa_family_t     sin_family;
	uint16_t        sin_port;
	struct in_addr  sin_addr;
	char            sin_zero[8];
};
```
### (1) 成员sin_family
地址族，每种协议族适用的地址族均不同。

|名称|地址族|
| :------: | :------: |
|AF_INET	|	IPv4网络协议使用的地址族|
|AF_INET6	|	IPv6网络协议使用的地址族|
|AF_LOCAL   |   本地通信中采用的UNIX协议的地址族|

### (2) 成员sin_port
16位端口号，以网络字节序保存。
### (3) 成员sin_addr
保存32位地址信息，以网络字节序保存。
```c
struct in_addr
{
    in_addr_t s_addr;
};
```
### (4) 成员sin_zero
无特殊含义，是为了使sockaddr_in的大小与sockaddr结构体的大小一致而插入的，必须填充为0.
## 3. sockaddr_in结构体的使用
bind函数原型：
```c
bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);
```
第二个参数接期望得到一个sockaddr结构体变量地址值。
```c
struct sockaddr
{
	sa_family_t     sin_family; 
	char sa_data[14];				/* Address data.  */
};
```
所以使用时要将sockaddr_in结构体变量的地址转换位sockaddr结构体变量的指针，然后再传递给bind函数。
```c
struct sockaddr_in serv_addr;
…………
if(bind(serv_sock, (struct sockaddr*) &serv_addr, sizeof(serv_addr))==-1)
		error_handling("bind() error"); 
…………
```

## 4. 网络字节序与地址变换
- 字节序有大端小端之分。
    - 大端模式，是指数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址中。
    - 小端模式，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中。
- 网络字节序统一为大端方式。所以传输之前应该进行字节序转换。
- 为了进行转换socket提供了转换的函数有下面四个:
```c
#include <arpa/inet.h>			\\头文件
unsigned short htons(unsigned short); 	\\把unsigned short类型从主机序转换到网络序
unsigned short ntohs(unsigned short); 	\\把unsigned short类型从网络序转换到主机序
unsigned long htonl(unsigned long); 	\\把unsigned long类型从主机序转换到网络序
unsigned long ntohl(unsigned long);	\\把unsigned long类型从网络序转换到主机序
```
- 示例：

```c
#include <stdio.h>
#include <arpa/inet.h>

int main()
{
	unsigned short host_port=0x1234;
	unsigned short net_port;
	unsigned long host_addr=0x12345678;
	unsigned long net_addr;
	
	net_port = htons(host_port);
	net_addr = htons(host_addr);
	
	printf("Host ordered port: %#x \n", host_port);
	printf("Network ordered port: %#x \n", net_port);
	printf("Host ordered address: %#lx \n", host_addr);
	printf("Network ordered address: %#lx \n", net_addr);

	return 0;
}

/**
output:
Host ordered port: 0x1234 
Network ordered port: 0x3412 
Host ordered address: 0x12345678 
Network ordered address: 0x7856
 */
```

## 5. 网络地址的初始化与分配

### (1) 将字符串信息转换为网络字节序的整数型值

**inet_addr()函数**
- 头文件：<arpa/inet.h>
- 原型：
```c
in_addr_t inet_addr (const char *string);
```
- 参数：点分十进制格式的IP地址字符串
- 返回值： 成功返回32位大端序整数型值，失败返回INADDR_NONE。

**inet_aton()函数**
- 头文件：<arpa/inet.h>
- 原型：
```c
int inet_aton (const char * string, struct in_addr *addr);
```
- 参数：
    - string---点分十进制格式的IP地址字符串
    - addr---保存转换结果的in_addr结构体变量的地址值。
- 返回值：成功返回1，失败返回0

**inet_ntoa()函数** 
- 头文件：<arpa/inet.h>
- 原型：
```c
char *inet_ntoa (struct in_addr adr);
```
- 参数：需要转换的in_addr结构体
- 返回值：成功时返回转换的字符串地址值，失败返回-1

**示例代码:**
```c
#include <stdio.h>
#include <arpa/inet.h>

int main()
{
	char *addr1="1.2.3.4";
	char *addr2="127.212.124.256";
	
	unsigned long conv_addr = inet_addr(addr1);
	if(conv_addr==INADDR_NONE)
		printf("Error occured! \n");
	else
		printf("Network ordered integer addr: %#lx \n", conv_addr);
	
	conv_addr = inet_addr(addr2);
	if(conv_addr==INADDR_NONE)
		printf("Error occureded \n");
	else
		printf("Network ordered integer addr: %#lx \n\n", conv_addr);
	return 0;
}

/**
output:
Network ordered integer addr: 0x4030201 
Error occureded
 */
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>

int main()
{
	char * addr = "1.2.3.4";
	struct sockaddr_in addr_inet;
	
	if(!inet_aton(addr,&addr_inet.sin_addr))
		printf("Conversion error");
	else 
		printf("Network ordered integer addr: %#x \n",addr_inet.sin_addr.s_addr);

	return 0;
}

/**
output:
Network ordered integer addr: 0x4030201
 */
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>

int main()
{
	struct sockaddr_in addr;
	char * str_ptr;
	addr.sin_addr.s_addr = htonl(0x1020304);
	str_ptr= inet_ntoa(addr.sin_addr);
	
	printf("Dotted-Decimal notation: %s \n",str_ptr);
	return 0;
}

/**
output:
Dotted-Decimal notation: 1.2.3.4 
*/
```
### (2) 常见的套接字创建过程中网络地址信息初始化方法
```
struct sockaddr_in addr;
char * serv_ip ="211.217.168.13";
char * serv_port = "9190";
memset(&addr, 0, sizeof(addr));
addr.sin_family=AF_INET;
addr.sin_addr.s_addr=htonl(serv_ip);
addr.sin_port=htons(atoi(serv_port));
```
可以利用INADDR_ANY常量分配服务端IP地址，自动获取本机IP。

### (3) 向套接字分配网络地址 bind()函数
- 功能：
    bind函数将socket与网络地址信息绑定。
- 原型：
```c
int bind (int sockfd, struct sockaddr *myaddr, socklen_t addrlen);
```
- 参数：
    - sockfd---要分配地址信息的套接字文件描述符
	- myaddr---存有地址信息的sockaddr结构体变量的地址值。
	- addrlen---第二个结构体变量的长度
- 返回值：
    成功返回0，失败返回-1

# 第四、五章 基于TCP的服务端/客户端
## 1. TCP服务器端的默认函数调用顺序
- socket() 创建套接字
- bind() 分配给套接字地址
- listen() 等待连接请求状态
- accept() 允许连接，接收新连接
- read()/write() 数据交换
- close() 关闭套接字，断开连接

## 2. 进入等待连接请求状态 listen()函数
- 功能：
    进入等待连接请求状态
- 原型：
```
listen(int sock, int backlog);
```
- 参数：
    - sock---希望进入等待连接请求状态的套接字文件描述符，该套接字将变为服务端套接字即监听套接字。
    - backlog---连接请求等待队列的长度。
- 返回值：
    成功返回0，失败返回-1

## 3. 受理客户端连接请求 connect()函数
- 功能：接受客户端连接请求
- 原型：
```c
int accept (int sock, struct sockaddr * addr, socklen_t *addrlen);
```
- 参数：
    - sock---服务端套接字文件描述符
    - addr---保存发起连接请求的客户端地址信息的变量地址值，函数调用完成后，该地址保存的是客户端地址信息
    - addrlen---保存第二个参数sockaddr结构体的长度，函数调用完成后，该地址保存的客户端地址信息的长度
- 返回值：成功时返回创建的套接字文件描述符，用以数据交换，失败时返回-1

## 4. TCP客户端的默认函数调用顺序
- socket() 创建套接字
- connect() 请求连接
- read()/write() 数据交换
- close() 关闭套接字，断开连接

## 5. 客户端发起连接请求connect()函数
- 功能：
    请求连接到服务器
- 原型：
```
int connect(int sock, struct sockaddr * servaddr, socklen_t addrlen);
```
- 参数：
    - sock---客户端套接字文件描述符
	- servaddr---存有目标服务器端地址信息的变量地址值
	- addrlen---第二个参数sockaddr结构体的长度

- 客户端调用connect函数后，发生一下情况之一才会返回：
1. 服务端接受连接请求。
2. 发生断网等异常情况而中断连接请求。
- 注：客户端的IP地址和端口号在调用connect函数是自动分配。

## 6. 第一个基于TCP的服务端/客户端---HelloWorld服务器
服务端代码：
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
//错误处理
void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}

int main(int argc,char *argv[])
{
	//定义套接字
	int serv_sock; //用于监听的套接字
	int clnt_sock; //用于数据传输的套接字
	//定义地址信息
	struct sockaddr_in serv_addr; //服务端地址信息
	struct sockaddr_in clnt_addr; //请求连接的客户端地址信息
	socklen_t clnt_addr_size;	//客户端地址信息长度

	char message[]="Hello World!";
	if(argc!=2)
	{
		printf("Usage : %s <port>\n",argv[0]);
		exit(1);
	}
	//创建套接字
	serv_sock = socket(PF_INET,SOCK_STREAM,0);
	if(serv_sock == -1)
		error_handling("socket() error");
	//地址信息初始化
	memset(&serv_addr,0,sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
	serv_addr.sin_port = htons(atoi(argv[1]));
	//绑定地址信息
		//需要将sockaddr_in结构体变量地址强制转换为sockaddr结构体类型指针
	if(bind(serv_sock,(struct sockaddr*) &serv_addr,sizeof(serv_addr))==-1) 
		error_handling("bind() error");
	//监听
	if(listen(serv_sock,5)==-1)
		error_handling("liste() error");
	//接受客户端连接请求
		//需要将sockaddr_in结构体变量地址强制转换为sockaddr结构体类型指针
	clnt_addr_size = sizeof(clnt_addr);
	clnt_sock = accept(serv_sock,(struct sockaddr*) &clnt_addr,&clnt_addr_size);
	if(clnt_sock == -1)
		error_handling("accept() error");
	//向客户端发送一句“Hello World!”
	write(clnt_sock,message,sizeof(message));
	//关闭数据交换套接字
	close(clnt_sock);
	//关闭监听套接字
	close(serv_sock);
	return 0;
}

```
客户端
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char * message)
{
	fputs(message,stderr);
	fputc('\n',stderr);
	exit(1);
}

int main(int argc,char *argv[])
{
	//定义套接字
	int sock;
	//定义服务端地址信息
	struct sockaddr_in serv_addr;
	//存放接受的消息
	char message[30];
	if(argc!=3)
	{
		printf("Usage : %s <IP> <port>\n",argv[0]);
		exit(1);
	}
	//创建套接字
	sock = socket(PF_INET,SOCK_STREAM,0);
	if(sock==-1)
		error_handling("socket() error");
	//服务端地址信息初始化
	memset(&serv_addr,0,sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
	serv_addr.sin_port = htons(atoi(argv[2]));
	//请求连接服务端
		//需要将sockaddr_in结构体变量地址强制转换为sockaddr结构体类型指针
	if(connect(sock,(struct sockaddr *) &serv_addr,sizeof(serv_addr))==-1)
		error_handling("conect() error!");
	//接受服务端发送的信息
	int str_len = read(sock,message,sizeof(message)-1);
	if(str_len == -1)
		error_handling("read() error!");
	
	printf("Message from server : %s \n",message);
	//关闭套接字
	close(sock);
	return 0;
}

```
## 7. 基于TCP的服务端客户端函数调用关系
服务端创建套接字后连续调用bind(),listen()函数进入等待状态，客户端创建套接字后通过connect()发起连接请求。服务端调用accept(),接受一个连接请求，接受之后客户端connect函数返回，双方进入数据交换阶段，如果队列中没有连接请求，则进入阻塞状态。数据交换阶段，通过调用read()/write()函数交换数据。结束之后调用close()关闭套接字，断开连接。

## 8. 迭代服务端/客户端

### (1) 迭代服务端函数调用顺序
重复调用accept函数以实现可以向多个客户端提供服务但是统一时刻还是只能为一个客户端提供服务，与一个客户端通信是其他客户端只能阻塞。
### (2) 迭代的回声服务端/客户端
```c
…………
for(i=0; i<5; i++)
	{
		clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
		if(clnt_sock==-1)
			error_handling("accept() error");
		else
			printf("Connected client %d \n", i+1);
	
		while((str_len=read(clnt_sock, message, BUF_SIZE))!=0)
			write(clnt_sock, message, str_len);

		close(clnt_sock);
	}
…………
```
## 9. 解决TCP不存在数据边界的问题
自定义应用层协议
## 10. TCP原理
### (1) I/O缓冲
- I/O缓冲在每个TCP套接字中单独存在
- I/O缓冲在创建套接字时自动生成
- 即使关闭套接字也会继续传递输出缓冲中遗留的数据
- 关闭套接字将丢失输入缓冲中的数据

### (2) 套接字的连接
三次握手

**套接字是以全双工的方式工作的，也就是说，它可以双向传递数据。**

### (3) 数据交换

### (4) 断开套接字连接
四次挥手

# 第六章 基于UDP的服务端/客户端
## 1. UDP协议
- TCP比UDP慢的原因
    - 收发数据前后需要进行连接和清除过程
    - 收发数据过程中需要添加流控制, 以保证数据可靠性
- UDP的高效使用
    - 如果数据完整性要求高, 使用TCP, 例如传送压缩的数据
    - 如果可以容忍少量数据丢失, 使用UDP, 例如视频音频

## 2. 实现基于UDP的服务端/客户端
- UDP服务端和客户端没有连接
    - 不是面向连接的, 不需要连接过程
    - 只需创建套接字, 然后进行数据交换, 不需要listen和accept
- UDP服务端和客户端均只需一个套接字
    - TCP: 用于连接的套接字,以及数据交换的套接字(将保持连接)
    - UDP: 只需一个套接字(每次交换数据是需要添加目标地址信息)

## 3. 基于UDP的数据I/O函数
**sendto函数**
- 功能： 
    发送数据到对端
- 原型：
```cpp
ssize_t sendto (int sock, const void *buff, size_t nbytes,
		       int flags, struct sockaddr *to,socklen_t addrlen);
```
- 参数：
    - sock---用于传输数据的套接字文件描述符
    - buff---存有待传送数据的缓冲地址值
    - nbytes---待传输数据长度
    - flags---可选参数，若没有传递0
    - to---存有目标地址信息的sockaddr结构体变量的地址值
    - addrlen---参数to的地址值结构体变量长度

**recvfrom函数**
- 功能：
    接收对端数据
- 原型：
```cpp
ssize_t recvfrom (int sock, void *buff, size_t nbytes,
			 int flags, struct sockaddr * from,socklen_t *addrlen);
```
- 参数：
    - sock---用于接受数据的套接字文件描述符
    - buff---用于保存接收数据的缓冲地址值
    - nbytes---可接收的最大字节数
    - flags---可选参数
    - from---存有发送端地址信息的sockaddr结构体变量的地址值
    - addrlen---参数法from的地址值结构体变量长度

**基于UDP的回声服务端/客户端:**

服务端
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30

void error_handling(char * message)
{
	fputs(message,stderr);
	fputc('\n',stderr);
	exit(1);
}

int main(int argc,char *argv[])
{
	int serv_sock;
	struct sockaddr_in serv_addr,clnt_addr;
	socklen_t clnt_addr_size;
	
	char message[BUF_SIZE];
	
	if(argc!=2)
	{
		printf("Usage : %s <port>\n",argv[0]);
		exit(1);
	}
	//创建套接字
	serv_sock=socket(PF_INET,SOCK_DGRAM,0);
	if(serv_sock==-1)
		error_handling("UDP socket() error");
	//初始化地址信息
	memset(&serv_addr,0,sizeof(serv_addr));
	serv_addr.sin_family=AF_INET;
	serv_addr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_addr.sin_port = htons(atoi(argv[1]));
	//绑定地址信息
	if(bind(serv_sock,(struct sockaddr *) &serv_addr,sizeof(serv_addr))==-1)
		error_handling("bind() error");
	//读写数据
	while(1)
	{
		clnt_addr_size = sizeof(clnt_addr);
		int str_len = recvfrom(serv_sock,message,sizeof(message),
				0,(struct sockaddr *) &clnt_addr,&clnt_addr_size);
		sendto(serv_sock,message,str_len,
				0,(struct sockaddr *) &clnt_addr,clnt_addr_size);
	}
	//关闭套接字
	close(serv_sock);
	return 0;
}

```
客户端
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 30

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}

int main(int argc,char *argv[])
{
	int sock;
	struct sockaddr_in serv_addr;
	socklen_t addr_size;
	
	char message[BUF_SIZE];
	

	if(argc!=3)
	{
		printf("Usage : %s <IP> <port>\n",argv[0]);
		exit(1);
	}
	//创建套接字
	sock = socket(PF_INET,SOCK_DGRAM,0);
	if(sock==-1)
		error_handling("socket() error");
	//初始化服务器地址信息
	memset(&serv_addr,0,sizeof(serv_addr));
	serv_addr.sin_family=AF_INET;
	serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
	serv_addr.sin_port = htons(atoi(argv[2]));
	//数据交换
	while(1)
	{
		fputs("Insert message(q to quit): ",stdout);
		fgets(message, sizeof(message),stdin);

		if(!strcmp(message,"q\n") || !strcmp(message,"Q\n"))
			break;
		
		sendto(sock,message,strlen(message),
				0,(struct sockaddr *) &serv_addr,sizeof(serv_addr));
		addr_size = sizeof(serv_addr);
		int str_len = recvfrom(sock,message,BUF_SIZE,
				0,(struct sockaddr *) &serv_addr,&addr_size);
		message[str_len] = 0;
		
		printf("Message from server:  %s",message);
	}
	//关闭套接字
	close(sock);
	return 0;
}

```
**注：UDP客户端套接字的地址分配：调用sendto函数时自动分配IP和端口号。**

## 4. UDP的数据传输特性
UDP具有数据边界，传输过程中调用I/O函数的次数必须完全一致。
因为存在数据边界, 一个数据报即可成为一个完整数据,所以UDP套接字传输的数据包又称为数据报。
## 5. 未连接UDP套接字与已连接UDP套接字
- UDP套接字默认都是未连接套接字,每次sendto都要经历三个阶段：
    - 向UDP套接字注册目标IP和端口号
    - 交换数据
    - 删除UDP套接字中注册的目标地址信息
- 如果连续想同一个IP地址和端口发送数据，未连接套接字浪费
- 已连接UDP套接字
创建已连接UDP套接字只需要针对套接字调用connect函数即可。之后可以使用sendto，recvfrom函数进行数据交换，还可以使用write，read函数交换数据。
- 示例：
```c
connect(sock,(struct sockaddr *) &serv_addr,sizeof(serv_addr));
while(1)
{
	fputs("Insert message(q to quit): ",stdout);
	fgets(message, sizeof(message),stdin);

	if(!strcmp(message,"q\n") || !strcmp(message,"Q\n"))
		break;
	
	/*
	sendto(sock,message,strlen(message),
			0,(struct sockaddr *) &serv_addr,sizeof(serv_addr));
	addr_size = sizeof(serv_addr);
	int str_len = recvfrom(sock,message,BUF_SIZE,
			0,(struct sockaddr *) &serv_addr,&addr_size);
	*/

	write(sock,message,strlen(message));
	int str_len=read(sock,message,BUF_SIZE);

	message[str_len] = 0;
	
	printf("Message from server:  %s",message);
}
```
# 第七章 优雅地断开套接字连接
## 1. 基于TCP的半关闭
半关闭：只关闭一部分数据交换中使用的流，即只关闭一个方向的数据交换。
## 2. 针对半关闭的shutdown()函数
- 原型：
```cpp
int shutdown (int sock, int howto);
```
- 参数：
    - sock --- 要断开的套接字文件描述符
    - howto --- 断开方式
    
|howto值|断开方式|
|:---:|:---:|
|SHUT_RD|断开输入流|
|SHUT_WR|断开输出流|
|SHUT_RDWR|同时断开I/O流|

- 返回值：成功返回0,失败返回-1

# 第八章 域名及网络地址
## 1. 域名系统
- DNS是对IP地址和域名进行转换的系统，其核心是DNS服务器。

## 2. 利用域名获取IP地址

**gethostbyname函数**
- 包含头文件：<netdb.h>
- 原型：
```c
struct hostent *gethostbyname (const char *hostname);
```
- 参数：
    hotname --- 域名字符串
- 返回值：
    成功时返回hostent结构体地址，失败时返回NULL指针
 
**hostent结构体**

```c
struct hostent
{
  char *h_name;			/* Official name of host.  */
  char **h_aliases;		/* Alias list.  */
  int h_addrtype;		/* Host address type.  */
  int h_length;			/* Length of address.  */
  char **h_addr_list;		/* List of addresses from name server.  */
};
```
- h_name：官方域名
- h_aliases：多个域名列表
- h_addrtype：地址类型，若是IPv4，则词变量为AF_INET
- h_length：IP地址长度。若是IPv4，为4,若是IPv6，为16
- h_addr_list：以整数形式保存的域名对应的IP地址

## 3. 利用IP地址获取域名

**gethostbyaddr函数**
- 包含头文件：<netdb.h>
- 原型：
```c
struct hostent *gethostbyaddr (const void *addr, socklen_t len, int family);
```
- 参数：
    - addr --- 含有IP地址信息的in_addr结构体指针。为了同时传递IPv4地址之外的信息，该变量的类型声明为char型指针。
    - len --- 第一个参数地址信息的字节数，IPv4为4,IPv6为16
    - family --- 地址族信息，IPv4时为AF_INET，IPv6时为AF_INET6
- 示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

int main(int argc,char * argv[])
{
	int i;
	struct hostent * host;
	host = gethostbyname(argv[1]);
	printf("Official name: %s \n", host->h_name);
	for(i=0;host->h_aliases[i];i++)
		printf("Aliases %d: %s \n", i+1, host->h_aliases[i]);
	printf("Address type: %s \n",
			(host->h_addrtype==AF_INET)?"AF_INET":"AF_INET6");
	for(i=0;host->h_addr_list[i];i++)
		printf("IP addr %d : %s \n",i+1,
				inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));
	return 0;
}

/*
output:
$./gethostbyname baidu.com
Official name: baidu.com 
Address type: AF_INET 
IP addr 1 : 180.149.132.47 
IP addr 2 : 220.181.57.217 
IP addr 3 : 111.13.101.208 
IP addr 4 : 123.125.114.144 
 */
```
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

int main(int argc,char * argv[])
{
	int i;
	struct hostent * host;
	struct sockaddr_in addr;
	
	memset(&addr,0,sizeof(addr));
	addr.sin_addr.s_addr = inet_addr(argv[1]);
	host = gethostbyaddr((char*)&addr.sin_addr,4,AF_INET);

	printf("Official name: %s \n", host->h_name);
	for(i=0;host->h_aliases[i];i++)
		printf("Aliases %d: %s \n", i+1, host->h_aliases[i]);
	printf("Address type: %s \n",
			(host->h_addrtype==AF_INET)?"AF_INET":"AF_INET6");
	for(i=0;host->h_addr_list[i];i++)
		printf("IP addr %d : %s \n",i+1,
				inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));
	return 0;
}
/*
output:
$ ./gethostbyaddr 127.0.0.1
Official name: localhost 
Address type: AF_INET 
IP addr 1 : 127.0.0.1 
 */
```

# 第九章 套接字的多种选项

## 1. 套接字选项

**getsockopt()**
- 头文件 `<sys/socket.h>`
- 功能：获取套接字选项
- 原型：
```c
int getsockopt (int sock, int level, int optname,
		       void *optval, socklen_t *optlen);
```
- 参数：
    - sock --- 用于查看套接字选项的套接字文件描述符
    - level --- 要查看选项的协议层
    - optname --- 要查看的选项名
    - optval --- 保存结果的缓冲地址值
    - optlen --- 保存通过第四个参数返回的可选项信息的字节数
- 返回值
    成功返回0,失败返回-1

**setsockopt()**
- 头文件 `<sys/socket.h>`
- 原型：
```c
int setsockopt (int sock, int level, int optname,
		       const void *optval, socklen_t optlen);
```
- 参数：
    - sock --- 要设置可选项的套接字文件描述符
    - level --- 要设置的选项的协议层
    - optname --- 要设置的选项名
    - optval --- 存有要设置的选项信息的缓冲地址值
    - optlen --- 向参数optval传递的选项信息的字节数
- 返回值：
    成功返回0,失败返回-1

## 2. I/O缓冲大小选项 SO_SNDBUF & SO_RECVBUF
- SO_RECVBUF是输入缓冲大小可选项
- SO_SNDBUF是输出缓冲大小可选项
- 示例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>

int main()
{
	int sock;
	int snd_buf,rcv_buf,state;
	socklen_t len;
	
	sock = socket(PF_INET,SOCK_STREAM,0);
	len = sizeof(snd_buf);
	state = getsockopt(sock,SOL_SOCKET,SO_SNDBUF,(void *)&snd_buf,&len);
	
	len = sizeof(snd_buf);
	state = getsockopt(sock,SOL_SOCKET,SO_RCVBUF,(void *)&rcv_buf,&len);
	
	printf("Input buffer size: %d \n", rcv_buf);
	printf("Outupt buffer size: %d \n", snd_buf);
	
	snd_buf /= 4;
	rcv_buf /= 4;
	
	state = setsockopt(sock,SOL_SOCKET,SO_SNDBUF,(void *)&snd_buf,sizeof(snd_buf));
	state = setsockopt(sock,SOL_SOCKET,SO_RCVBUF,(void *)&rcv_buf,sizeof(rcv_buf));

	len = sizeof(snd_buf);
	state = getsockopt(sock,SOL_SOCKET,SO_SNDBUF,(void *)&snd_buf,&len);
	
	len = sizeof(snd_buf);
	state = getsockopt(sock,SOL_SOCKET,SO_RCVBUF,(void *)&rcv_buf,&len);

	printf("Input buffer size: %d \n", rcv_buf);
	printf("Outupt buffer size: %d \n", snd_buf);
	
	return 0;
}

/*
output:
Input buffer size: 87380 
Outupt buffer size: 16384 
Input buffer size: 43690 
Outupt buffer size: 8192 
 */
```
## 3. 地址再分配选项 SO_REUSEADDR 和Time-wait状态

**Time-wait状态**
- Time-wait状态出现在主动断开连接的一方，即先调用close()的一方
- Time-wait状态的设置是为了确保最后一条ACK消息的准确到达
- Time-wait并非只有优点

**SO_REUSEADDR选项**

SO_REUSEADDR默认设置为0,意味着无法分配Time-wait状态下的套接字端口号。将该选项设置为1,则可将Time-wait状态下的套接字端口号重新分配给新的套接字。

## 4. Nagle算法选项 TCP_NODELAY

**Nagle算法**
- TCP默认使用Nagle算法交换数据，因此最大限度地进行缓冲，直到收到ACK。也就是只有收到前一数据的ACK消息时，才发送下一数据。
- 根据传输数据的特性，在网络流量未受太大影响时，不使用Nagle算法要比使用它时传输速度快。典型的是“传输大文件数据”。

**禁用Nagle算法**
- 如果有必要可以禁用Nagle算法
- 将TCP_NODELAY改为1即可禁用Nagle算法。