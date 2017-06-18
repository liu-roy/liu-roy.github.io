---
title: C++实现局域网IP多播
date: 2015-12-18 10:54:30
categories: 网络
tags: [自发现,多播]
---

 
转自http://www.51cto.com/specbook/17/35216.htm

在局域网中，管理员常常需要将某条信息发送给一组用户。如果使用一对一的发送方法，虽然是可行的，但是过于麻烦，也常会出现漏发、错发。为了更有效的解决这种组通信问题，出现了一种多播技术（也常称为组播通信），它是基于IP层的通信技术。为了帮助读者理解，下面将简要的介绍一下多播的概念。众所周知，普通IP通信是在一个发送者和一个接收者之间进行的，我们常把它称为点对点的通信，但对于有些应用，这种点对点的通信模式不能有效地满足实际应用的需求。例如：一个数字电话会议系统由多个会场组成，当在其中一个会场的参会人发言时，要求其它会场都能即时的得到此发言的内容，这是一个典型的一对多的通信应用，通常把这种一对多的通信称为多播通信。采用多播通信技术，不仅可以实现一个发送者和多个接收者之间进行通信的功能，而且可以有效减轻网络通信的负担，避免资源的无谓浪费。
广播也是一种实现一对多数据通信的模式，但广播与多播在实现方式上有所不同。广播是将数据从一个工作站发出，局域网内的其他所有工作站都能收到它。这一特征适用于无连接协议，因为LAN上的所有机器都可获得并处理广播消息。使用广播消息的不利之处是每台机器都必须对该消息进行处理。多播通信则不同，数据从一个工作站发出后，如果在其它LAN上的机器上面运行的进程表示对这些数据“有兴趣”，多播数据才会制给它们。

本实例由Sender和Receiver两个程序组成，Sender用户从控制台上输入多播发送数据，Receiver端都要求加入同一个多播组，完成接收Sender发送的多播数据。

## 一、实现方法

### 1、协议支持

并不是所有的协议都支持多播通信，对Win32平台而言，仅两种可从WinSock内访问的协议（IP/ATM）才提供了对多播通信的支持。因通常通信应用都建立在TCP/IP协议之上的，所以本文只针对IP协议来探讨多播通信技术。
支持多播通信的平台包括Windows CE 2.1、Windows 95、Windows 98、Windows NT 4、Windows 2000和WindowsXP。自2.1版开始，Windows CE才开始实现对IP多播的支持。本文实例建立在WindowsXP专业版平台上。

### 2、多播地址

IP采用D类地址来支持多播。每个D类地址代表一组主机。共有28位可用来标识小组。所以可以同时有多达25亿个小组。当一个进程向一个D类地址发送分组时，会尽最大的努力将它送给小组的所有成员，但不能保证全部送到。有些成员可能收不到这个分组。举个例子来说，假定五个节点都想通过I P多播，实现彼此间的通信，它们便可加入同一个组地址。全部加入之后，由一个节点发出的任何数据均会一模一样地复制一份，发给组内的每个成员，甚至包括始发数据的那个节点。D类I P地址范围在244.0.0.0到239.255.255.255之间。它分为两类：永久地址和临时地址。永久地址是为特殊用途而保留的。比如，244.0.0.0根本没有使用（也不能使用），244.0.0.1代表子网内的所有系统（主机），而244.0.0.2代表子网内的所有路由器。在RFC 1700文件中，提供了所有保留地址的一个详细清单。该文件是为特殊用途保留的所有资源的一个列表，大家可以找来作为参考。"Internet分配数字专家组"（I A N A）负责着这个列表的维护。在表1中，我们总结了目前标定为"保留"的一些地址。临时组地址在使用前必须先创建，一个进程可以要求其主机加入特定的组，它也能要求其主机脱离该组。当主机上的最后一个进程脱离某个组后，该组地址就不再在这台主机中出现。每个主机都要记录它的进程当前属于哪个组。表1部分永久地址说明：

地 址	说 明
244.0.0.1	基本地址（保留）
244.0.0.1	子网上的所有系统
244.0.0.2	子网上的所有路由器
244.0.0.5	子网上所有OSPF路由器
244.0.0.6	子网上所有指定的OSPF路由器
244.0.0.9	RIP第2版本组地址
244.0.1.1	网络时间协议
244.0.1.24	WINS服务器组地址

### 3、多播路由器

多播由特殊的多播路由器来实现，多播路由器同时也可以是普通路由器。各个多播路由器每分钟发送一个硬件多播信息给子网上的主机(目的地址为244.0.0.1)，要求它们报告其进程当前所属的是哪一组，各主机将它感兴趣的D类地址返回。这些询问和响应分组使用IGMP（Internet group management protocol），它大致类似于ICMP。它只有两种分组：询问和响应，都有一个简单的固定格式，其中有效载荷字段的第一个字段是一些控制信息，第二字段是一个D类地址，在RFC1112中有详细说明。
多播路由器的选择是通过生成树实现的，每个多播路由器采用修改过的距离矢量协议和其邻居交换信息，以便向每个路由器为每一组构造一个覆盖所有组员的生成树。在修剪生成树及删除无关路由器和网络时，用到了很多优化方法。

### 4、库支持

WinSock提供了实现多播通信的API函数调用。针对IP多播，WinSock提供了两种不同的实现方法，具体取决于使用的是哪个版本的WinSock。第一种方法是WinSock1提供的，要求通过套接字选项来加入一个组；另一种方法是WinSock2提供的，它是引入一个新函数，专门负责多播组的加入，这个函数便是WSAJoinLeaf，它是基层协议是无关的。本文将通过一个多播通信的实例的实现过程，来讲叙多播实现的主要步骤。因为Window98以后版本都安装了Winsock2.0以上版本，所以本文实例在WinSock2.0平台上开发的，但在其中对WinSock1实现不同的地方加以说明。

## 二、编程步骤

### 1、启动Visual C++6.0，创建一个控制台项目工程MultiCase。在此项目工程中添加Sender和Receiver两个项目。

Receiver项目实现步骤：
>* (1)、创建一个SOCK_DGRAM类型的Socket。
>* (2)、将此Socket绑定到本地的一个端口上，为了接收服务器端发送的多播数据。
>* (3)、加入多播组。
>* (4)、接收多播数据

####①、WinSock2中引入一个WSAJoinLeaf，此函数原型如下：
```c++

SOCKET WSAJoinLeaf( SOCKET s, const struct sockaddr FAR *name, int namelen, 
LPWSABUF lpCallerData, LPWSABUF lpCalleeData, LPQOS lpSQOS, 

        LPQOS lpGQOS, DWORD dwFlags );
```

其中，第一个参数s代表一个套接字句柄，是自WSASocket返回的。传递进来的这个套接字必须使用恰当的多播标志进行创建；否则的话WSAJoinLeaf就会失败，并返回错误WSAEINVAL。第二个参数是SOCKADDR（套接字地址）结构，具体内容由当前采用的协议决定，对于IP协议来说，这个地址指定的是主机打算加入的那个多播组。第三个参数namelen（名字长度）是用于指定name参数的长度，以字节为单位。第四个参数lpCallerData（呼叫者数据）的作用是在会话建立之后，将一个数据缓冲区传输给自己通信的对方。第五个参数lpCalleeData（被叫者数据）用于初始化一个缓冲区，在会话建好之后，接收来自对方的数据。注意在当前的Windows平台上，lpCallerData和lpCalleeData这两个参数并未真正实现，所以均应设为NULL。LpSQOS和lpGQOS这两个参数是有关Qos（服务质量）的设置，通常也设为NULL，有关Qos内容请参阅MSDN或有关书籍。最后一个参数dwFlags指出该主机是发送数据、接收数据或收发兼并。该参数可选值分别是：JL_SENDER_ONLY、JL_RECEIVER_ONLY或者JL_BOTH。

####②、在WinSock1平台上加入多播组需要调用setsockopt函数，同时设置IP_ADD_MEMBERSHIP选项，指定想加入的那个组的地址结构。具体实现代码将在下面代码注释列出。

### 2、Sender实现步骤：

(1)、创建一个SOCK_DGRAM类型的Socket。
(2)、加入多播组。
(3)、发送多播数据。

### 3、编译两个项目，在局域网中按如下步骤测试：

(1)、将Sender.exe拷贝到发送多播数据的ＰＣ上。
(2)、将Receiver.exe拷贝到多个要求接收多播数据的ＰＣ上。
(3)、各自运行相应的程序。
(4)、在Sender PC上输入多播数据后，你就可以在Receiver PC上看到输入的多播数据。

## 三、程序代码

### Receiver.c程序代码
```c++
#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>
#include <stdlib.h>
#define MCASTADDR "233.0.0.1" //本例使用的多播组地址。
#define MCASTPORT 5150 //绑定的本地端口号。
#define BUFSIZE 1024 //接收数据缓冲大小。
int main( int argc,char ** argv)
{
　WSADATA wsd;
　struct sockaddr_in local,remote,from;
　SOCKET sock,sockM;
　TCHAR recvbuf[BUFSIZE];
　/*struct ip_mreq mcast; // Winsock1.0 */

　int len = sizeof( struct sockaddr_in);
　int ret;
　//初始化WinSock2.2
　if( WSAStartup( MAKEWORD(2,2),&wsd) != 0 )
　{
　　printf("WSAStartup() failed\n");
　　return -1;
　}
　/*
　创建一个SOCK_DGRAM类型的SOCKET
　其中,WSA_FLAG_MULTIPOINT_C_LEAF表示IP多播在控制面层上属于"无根"类型;
　WSA_FLAG_MULTIPOINT_D_LEAF表示IP多播在数据面层上属于"无根",有关控制面层和
　数据面层有关概念请参阅MSDN说明。
　*/
　if((sock=WSASocket(AF_INET,SOCK_DGRAM,0,NULL,0,
　　WSA_FLAG_MULTIPOINT_C_LEAF|WSA_FLAG_MULTIPOINT_D_LEAF|
　　WSA_FLAG_OVERLAPPED)) == INVALID_SOCKET)
　{
　　printf("socket failed with:%d\n",WSAGetLastError());
　　WSACleanup();
　　return -1;
　}
　//将sock绑定到本机某端口上。
　local.sin_family = AF_INET;
　local.sin_port = htons(MCASTPORT);
　local.sin_addr.s_addr = INADDR_ANY;
　if( bind(sock,(struct sockaddr*)&local,sizeof(local)) == SOCKET_ERROR )
　{
　　printf( "bind failed with:%d \n",WSAGetLastError());
　　closesocket(sock);
　　WSACleanup();
　　return -1;
　}
　//加入多播组
　remote.sin_family = AF_INET;
　remote.sin_port = htons(MCASTPORT);
　remote.sin_addr.s_addr = inet_addr( MCASTADDR );
　/* Winsock1.0 */
　/*
　mcast.imr_multiaddr.s_addr = inet_addr(MCASTADDR);
　mcast.imr_interface.s_addr = INADDR_ANY;
　if( setsockopt(sockM,IPPROTO_IP,IP_ADD_MEMBERSHIP,(char*)&mcast,

            sizeof(mcast)) == SOCKET_ERROR)
　{
　　printf("setsockopt(IP_ADD_MEMBERSHIP) failed:%d\n",WSAGetLastError());
　　closesocket(sockM);
　　WSACleanup();
　　return -1;
　}
　*/
　/* Winsock2.0*/
　if(( sockM = WSAJoinLeaf(sock,(SOCKADDR*)&remote,sizeof(remote),

             NULL,NULL,NULL,NULL,
JL_BOTH)) == INVALID_SOCKET)
　{
　　printf("WSAJoinLeaf() failed:%d\n",WSAGetLastError());
　　closesocket(sock);
　　WSACleanup();
　　return -1;
　}
　//接收多播数据，当接收到的数据为"QUIT"时退出。
　while(1)
　{
　　if(( ret = recvfrom(sock,recvbuf,BUFSIZE,0,

            (struct sockaddr*)&from,&len)) == SOCKET_ERROR)
　　{
　　　printf("recvfrom failed with:%d\n",WSAGetLastError());
　　　closesocket(sockM);
　　　closesocket(sock);
　　　WSACleanup();
　　　return -1;
　　}
　　if( strcmp(recvbuf,"QUIT") == 0 ) break;
　　else {
　　　recvbuf[ret] = '\0';
　　　printf("RECV:' %s ' FROM <%s> \n",recvbuf,inet_ntoa(from.sin_addr));
　　}
　}

　closesocket(sockM);
　closesocket(sock);
　WSACleanup();
　return 0;
}
```


### Sender.c程序代码

```c++

#include <winsock2.h>
#include <ws2tcpip.h>
#include <stdio.h>
#include <stdlib.h>
#define MCASTADDR "233.0.0.1" //本例使用的多播组地址。
#define MCASTPORT 5150 //本地端口号。
#define BUFSIZE 1024 //发送数据缓冲大小。
int main( int argc,char ** argv)
{
　WSADATA wsd;
　struct sockaddr_in remote;
　SOCKET sock,sockM;
　TCHAR sendbuf[BUFSIZE];
　int len = sizeof( struct sockaddr_in);
　//初始化WinSock2.2
　if( WSAStartup( MAKEWORD(2,2),&wsd) != 0 )
　{
　　printf("WSAStartup() failed\n");
　　return -1;
　}
　if((sock=WSASocket(AF_INET,SOCK_DGRAM,0,NULL,0,
　　WSA_FLAG_MULTIPOINT_C_LEAF|WSA_FLAG_MULTIPOINT_D_LEAF|
　　WSA_FLAG_OVERLAPPED)) == INVALID_SOCKET)
　{
　　printf("socket failed with:%d\n",WSAGetLastError());
　　WSACleanup();
　　return -1;
　}
　//加入多播组
　remote.sin_family = AF_INET;
　remote.sin_port = htons(MCASTPORT);
　remote.sin_addr.s_addr = inet_addr( MCASTADDR );
　if(( sockM = WSAJoinLeaf(sock,(SOCKADDR*)&remote,
　　sizeof(remote),NULL,NULL,NULL,NULL,
　　JL_BOTH)) == INVALID_SOCKET)
　{
　　printf("WSAJoinLeaf() failed:%d\n",WSAGetLastError());
　　closesocket(sock);
　　WSACleanup();
　　return -1;
　}

　//发送多播数据，当用户在控制台输入"QUIT"时退出。
　while(1)
　{
　　printf("SEND : ");
　　scanf("%s",sendbuf);
　　if( sendto(sockM,(char*)sendbuf,strlen(sendbuf),0,(struct sockaddr*)

           &remote,sizeof(remote))==SOCKET_ERROR)
　　{
　　　printf("sendto failed with: %d\n",WSAGetLastError());
　　　closesocket(sockM);
　　　closesocket(sock);
　　　WSACleanup();
　　　return -1;
　　}
　　if(strcmp(sendbuf,"QUIT")==0) break;
　　Sleep(500);
　}

　closesocket(sockM);
　closesocket(sock);
　WSACleanup();
　return 0;
}
```


## 四、小结

本实例对IP多播通信进行了探讨，实例程序由Sender和Receiver两部分组成，Sender用户从控制台上输入多播发送数据，Receiver端都要求加入同一个多播组，完成接收Sender发送的多播数据。