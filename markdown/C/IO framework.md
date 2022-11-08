# 一. Windows系统

## 1. WinSock

```c++
#include <iostream>
#include <WinSock2.h>
#include <Ws2tcpip.h>

#pragma comment (lib,"ws2_32.lib")
#pragma warning(disable: 4996)

using namespace std;

#define BUF_SIZE 8192

// 获得本机IP
void GetLocalIP()
{
	int nLen = 20;
	char hostname[20];
	gethostname(hostname, nLen);
	cout << "主机名：" << hostname << endl;
	struct hostent FAR* lpHostEnt = gethostbyname(hostname);
	if (lpHostEnt == NULL) {
		cout << "IP：" << "0.0.0.0" << endl;
		return;
	}

	LPSTR lpAddr = lpHostEnt->h_addr_list[0];  // 取得IP地址列表中的第一个为返回的IP
	struct in_addr inAddr;
	memmove(&inAddr, lpAddr, 4);
	cout << "IP：" << inet_ntoa(inAddr) << endl;
}

int main()
{
	WSADATA wsaData;
	// MAKEWORD(a, b)：a在低位表示主版本，b在高位表示次版本
	int nRet = WSAStartup(MAKEWORD(2, 2), &wsaData); // 开启winsock.dll
	if (nRet != 0) {
		cout << "WSAStartup失败\n";
		return 0;
	}
	cout << "设置版本：" << (int)LOBYTE(wsaData.wVersion) << "." << (int)HIBYTE(wsaData.wVersion) << endl;
	cout << "最高版本：" << (int)LOBYTE(wsaData.wHighVersion) << "." << (int)HIBYTE(wsaData.wHighVersion) << endl;
	GetLocalIP();

	SOCKADDR_IN ServerAddr;
	ServerAddr.sin_family = AF_INET;
	ServerAddr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
	ServerAddr.sin_port = htons(5538);

	SOCKET sockListen = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	nRet = bind(sockListen, (LPSOCKADDR)&ServerAddr, sizeof(ServerAddr));
	if (nRet == SOCKET_ERROR) {
		cout << "bind失败\n";
		goto cleanup;
	}

	nRet = listen(sockListen, 10);
	if (nRet == SOCKET_ERROR)
	{
		cout << "listen失败\n";
		goto cleanup;
	}

	while (TRUE) {
		SOCKADDR_IN ClientAddr;
		int addr_length = sizeof(ClientAddr);
		SOCKET sockComm = accept(sockListen, (SOCKADDR*)&ClientAddr, &addr_length);
		if (sockComm == INVALID_SOCKET){
			cout << "accept失败\n";
			goto cleanup;
		}
		char ipAddress[INET_ADDRSTRLEN];
		inet_ntop(AF_INET, &ServerAddr.sin_addr, ipAddress, INET_ADDRSTRLEN);
		cout << "客户端IP：" << ipAddress << endl;

		while (TRUE)
		{
			char pRevMsg[BUF_SIZE] = { 0 };
			int iLen = recv(sockComm, pRevMsg, BUF_SIZE, 0);
			if (iLen > 0)
			{
				if (strcmp((LPCSTR)pRevMsg, "[SERVER EXIT]") == 0) {
					cout << "收到服务端退出通知" << endl;
					goto cleanup;
				}
				else if (strcmp((LPCSTR)pRevMsg, "[CLIENT EXIT]") == 0) {
					cout << "收到客户端退出通知" << endl;
					closesocket(sockComm);
					break;
				}
				else {
					cout << "收到消息：" << pRevMsg << endl;
				}
			}
			else if (iLen == SOCKET_ERROR)
			{
				cout << "recv失败" << endl;
				closesocket(sockComm);
				break;
			}
		}
	}
cleanup:
	closesocket(sockListen);
	WSACleanup();
	return 0;
}
```

## 2. select模型

### 1. select模型原理

每个客户端都有一个socket，服务器也有自己的socket，将所有的socket放进一个数据结构里。
通过select函数遍历装有socket数组的数据结构，当某个select有响应时，select就会通过其相应的参数值反馈出来。
在这里需要对返回值进行判断，如果返回值是服务器的socket，则是客户端请求连接，这时需要调用accept函数接收连接；如果返回值是客户端的socket，则是客户端请求通信，调用send函数或recv函数收发消息。

### 2. select模型用法

首先我们使用一个结构体用来装客户端的socket，系统已经为我们定义了一个fd_set结构体，结构体原型如下所示：

```c
typedef struct fd_set {
        u_int fd_count;               /* how many are SET? */
        SOCKET  fd_array[FD_SETSIZE];   /* an array of SOCKETs */
} fd_set;
```

  结构体中第一个成员fd_count是结构体成员的个数，第二个成员fd_array是一个socket类型的数组，系统将FD_SETSIZE利用宏定义定义为64，代表该数组最多有64个成员，也就是最多有64个客户端连接。
  同时系统为我们定义了四个fd_set的参数宏：

FD_ZERO,将结构体清零，FD_ZERO；
FD_SET，向结构体中添加一个socket，添加前会检查数组中元素是否超过64和数组中是否已经存在该元素；
FD_CLR：删除数组中指定的socket，从集合中删除一个socket后一定要closesocket，否则会造成内存泄露；
FD_ISSET：判断一个socket是否在集合中，若不存在返回0，若存在返回非零；
**select函数使用方法：**

```c
struct timeval {
        long    tv_sec;         /* seconds */
        long    tv_usec;        /* and microseconds */
};

// 如果客户端在等待时间内没有反应，返回0；
// 如果有客户端请求交流，返回一个大于零的数；
// 如果发生了错误则返回SOCKET_ERROR，通过WSAGetLastError()得到错误码。
int WSAAPI select{
		int nfds;
		fd_set *readfds;
		fd_set *writefds;
		fd_set *exceptfds;
		const timeval *timeout;
};
```

参数1：忽略（填0即可），这个参数仅仅是为了兼容Berkeley sockets；
参数2：检查是否有可读的socket，即客户端发来了消息，该socket就会被设置。 原理：它开始时包含所有的socket，通过select函数全部投递给系统，系统将有事件发生的socket再重新赋值给参数2，这样参数2就包含了所有有事件发生的socket了。
参数3：检查是否有可写的socket，就是可以给哪些客户端套接字发消息，即send，只要连接成功建立起来了，那该客户端套接字就是可写的（不一定非要在参数3中使用send方法）。 原理：它开始时包含所有的socket，通过select全部投放给系统，系统将可写的socket再赋值回来，调用后这个参数就是装着可以被send信息的客户端socket上。
检查套接字上的异常错误，用法跟参数2/3一样，将有异常错误的套接字重新装进来，反馈给我们。
参数5：最大等待时间，当客户端没有请求时，那么select函数可以等一会，如果我们设置的最大等待时间过后还没有请求，那就继续执行select下面的语句；如果在最大等待时间之内有请求，则立刻执行下面的语句。timeval结构体第一个参数tv_sec代表等待的秒数，第二个参数tv_usec代表等待的微秒数。如果两个参数均设置为0则select函数不会等待，如果参数5直接填NULL则select将完全阻塞，直至有事件响应才会向下执行（一般不要填NULL）。

### 3. 例子

服务端：

```c++
#define _CRT_SECURE_NO_WARNINGS
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <WinSock2.h>
#pragma comment(lib,"ws2_32.lib")
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

fd_set all_Sockets;

BOOL WINAPI over(DWORD dwCtrlType)
{
	switch (dwCtrlType)
	{
	case CTRL_CLOSE_EVENT:
		//释放所有socket
		for (u_int i = 0; i < all_Sockets.fd_count; i++)
		{
			closesocket(all_Sockets.fd_array[i]);
		}
		//清理网络库
		WSACleanup();
	}
	return 0;
}

int main()
{
	// 这个函数的作用是当点击运行框右上角叉号关闭时，执行上面的over函数
	SetConsoleCtrlHandler(over, TRUE);

	WSADATA wdSockMsg; // 系统通过这个参数给我们一些配置信息
	// 打开/启动网络库，只有启动了库，这个库里的函数才能使用
	int nRes = WSAStartup(MAKEWORD(2, 2), &wdSockMsg);
	if (0 != nRes)
	{
		switch (nRes)
		{
		case WSASYSNOTREADY:
			printf("可以重启电脑，或检查网络库");
			break;
		case WSAVERNOTSUPPORTED:
			printf("请更新网络库");
			break;
		case WSAEINPROGRESS:
			printf("请重新启动此软件");
			break;
		case WSAEPROCLIM:
			printf("请关闭不必要的软件，以为当前网络提供充足资源");
			break;
		case WSAEFAULT:
			printf("参数错误");
			break;
		}
	}
	//版本校验
	if (2 != HIBYTE(wdSockMsg.wVersion) || 2 != LOBYTE(wdSockMsg.wVersion))
	{
		// 版本打开错误， 关闭网络库
		WSACleanup();
		return 0;
	}

	// socket函数三个参数分别为地址类型（IPV4），套接字类型（）和协议类型（TCP） 
	// 如果执行失败则返回INVALID_SOCKET
	SOCKET socketSever = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (INVALID_SOCKET == socketSever)
	{
		// 如果socket调用失败，返回错误码（工具 -> 错误查找 可以查询具体错误）
		int a = WSAGetLastError();
		WSACleanup();
		return 0;
	}
	struct sockaddr_in si;
	si.sin_family = AF_INET;							//地址类型
	si.sin_port = htons(12345);							//端口号
	si.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");	//IP地址

	int bres = bind(socketSever, (const struct sockaddr*)&si, sizeof(si));
	if (SOCKET_ERROR == bres)
	{
		// bind函数出错
		int a = WSAGetLastError();		//返回错误码
		closesocket(socketSever);		//关闭socket
		WSACleanup();					//关闭网络库
		return 0;
	}
	//开始监听
	int a = listen(socketSever, SOMAXCONN);
	if (SOCKET_ERROR == a)
	{
		//listen 函数出错
		int a = WSAGetLastError();		//返回错误码
		closesocket(socketSever);		//关闭socket
		WSACleanup();					//关闭网络库
		return 0;
	}

	FD_ZERO(&all_Sockets);	//清零
	FD_SET(socketSever, &all_Sockets);//添加服务器socket

	while (1)
	{
		fd_set readSockets = all_Sockets;
		fd_set writeSockets = all_Sockets;
		fd_set errorSockets = all_Sockets;
		//时间段
		struct timeval timeval_a;//给参数5赋值等待时间
		timeval_a.tv_sec = 3;
		timeval_a.tv_usec = 0;
		int select_a = select(0, &readSockets, &writeSockets, &errorSockets, &timeval_a);
		//第二个参数测试recv和accept，第三个参数测试send，第四个参数测试错误
		if (0 == select_a)
		{
			//没有响应
			continue;
		}
		else if (select_a > 0)//有响应
		{
			//遍历参数4，查看select函数是否有错误返回
			for (u_int i = 0; i < errorSockets.fd_count; i++)
			{
				char str[100] = { 0 };
				int len = 99;
				if (SOCKET_ERROR == getsockopt(errorSockets.fd_array[i], SOL_SOCKET, SO_ERROR, str, &len))//调用getsockopt函数获取错误信息
					//参数1：我们要操作的socket，参数2：socket上的情况，参数4：代表一段空间，返回的错误信息装在里面，参数5：参数4的长度
				{
					printf("无法得到错误信息\n");
				}
				printf("%s\n", str);
			}
			// 遍历参数3，寻找找出可以给哪些客户端socket发消息
			for (u_int i = 0; i < writeSockets.fd_count; i++)
			{
				// printf("服务器：%d，%d可写\n", socketSever, writeSockets.fd_array[i]);
				if (SOCKET_ERROR == send(writeSockets.fd_array[i], "OK", 2, 0))
				{
					int a = WSAGetLastError();
				}
			}
			for (u_int i = 0; i < readSockets.fd_count; i++)
			{
				//遍历参数2中（有响应）的socket，在这里响应的socket只可能是服务器socket和客户端socket两种可能
				if (readSockets.fd_array[i] == socketSever)//如果有响应的socket是服务器socket，则是客户端请求连接，需要调用accept函数
				{
					//accept
					SOCKET socketClient = accept(socketSever, NULL, NULL);
					if (INVALID_SOCKET == socketClient)
					{
						continue;
					}
					FD_SET(socketClient, &all_Sockets); // 将刚返回的socket添加到socket数组中
				}
				else      //如果是客户端socket则需要接收消息
				{
					char buf[1500] = { 0 };
					int recv_a = recv(readSockets.fd_array[i], buf, 1500, 0);
					if (0 == recv_a)
					{
						printf("客户端下线\n");
						SOCKET socket_temp = readSockets.fd_array[i];
						//从集合中拿掉
						FD_CLR(readSockets.fd_array[i], &all_Sockets);
						//释放
						closesocket(socket_temp);
					}
					else if (0 < recv_a)
					{
						//接收成功
						printf("%s\n", buf);
					}
					else
					{
						//recv函数出错
						int a = WSAGetLastError();
					}
				}
			}
		}
		else
		{
			printf("错误码2：%d\n", WSAGetLastError());
		}
	}
	//释放所有socket
	for (u_int i = 0; i < all_Sockets.fd_count; i++)
	{
		closesocket(all_Sockets.fd_array[i]);
	}
	WSACleanup();				//关闭网络库
	return 0;
}
```

客户端：

```c++
#define _CRT_SECURE_NO_WARNINGS
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include<stdio.h>
#include<winsock2.h>
#pragma comment(lib,"Ws2_32.lib")
//#include<string.h>

int main()
{
	WORD wdVersion = MAKEWORD(2, 2);	//使用网络库的版本
	WSADATA wdSockMsg;					//系统通过这个参数给我们一些配置信息
	int nRes = WSAStartup(wdVersion, &wdSockMsg);

	if (0 != nRes)
	{
		switch (nRes)
		{
		case WSASYSNOTREADY:
			printf("可以重启电脑，或检查网络库");
			break;
		case WSAVERNOTSUPPORTED:
			printf("请更新网络库");
			break;
		case WSAEINPROGRESS:
			printf("Please reboot this software");
			break;
		case WSAEPROCLIM:
			printf("请关闭不必要的软件，以为当前网络提供充足资源");
			break;
		case WSAEFAULT:
			printf("参数错误");
			break;
		}
		return 0;
	}
	//版本校验
	if (2 != HIBYTE(wdSockMsg.wVersion) || 2 != LOBYTE(wdSockMsg.wVersion))
	{
		//版本打开错误
		WSACleanup();   //关闭网络库
		return 0;
	}

	//服务器的socket
	SOCKET socketSever = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);//这三个参数分别为地址类型（IPV4），套接字类型和协议类型（TCP)
																//如果执行失败则返回INVALID_SOCKET
	if (INVALID_SOCKET == socketSever)
	{
		int a = WSAGetLastError();				//如果socket调用失败，返回错误码（工具 -> 错误查找）
		WSACleanup();							//关闭网络库
		return 0;
	}

	struct sockaddr_in si;
	si.sin_family = AF_INET;
	si.sin_port = htons(12345);
	si.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");
	int connect_a = connect(socketSever, (const struct sockaddr*)&si, sizeof(si));
	if (SOCKET_ERROR == connect_a)
	{
		int a = WSAGetLastError();				//如果socket调用失败，返回错误码（工具 -> 错误查找）
		closesocket(socketSever);				//关闭socket
		WSACleanup();							//关闭网络库
		return 0;
	}
	send(socketSever, "连接成功", strlen("连接成功"), 0);//如果连接成功，向服务端发送“连接成功”

	while (1)
	{
		/*char buf[1500] = { 0 };
		int res = recv(socketSever, buf, 1499, 0);
		if (0 == res)
		{
			printf("连接中断，客户端下线\n");
		}
		else if (SOCKET_ERROR == res)
		{
			printf("错误码:%d\n", WSAGetLastError());
		}
		else
		{
			printf("%d,%s\n", res, buf);
		}*/

		//发送函数
		char buf[1500] = { 0 };
		scanf("%s", buf);
		if ('0' == buf[0])//输入0时，退出循环，客户端下线
		{
			break;
		}
		int send_a = send(socketSever, buf, strlen(buf), 0);
		if (SOCKET_ERROR == send_a)
		{
			//出现错误
			int a = WSAGetLastError();
		}
	}
	//关闭socket
	closesocket(socketSever);
	//清理网络库
	WSACleanup();
	return 0;
}

```

## 3. 异步选择模型：WSAAsyncSelect

### 1. 原理

核心：消息队列，操作系统为每个窗口创建一个消息队列并且维护，所以我们想要使用消息队列那就要创建一个窗口。
第一步：将我们的socket绑定到一个消息上，并投递给操作系统（WSAAsynSelect）
第二步：取出消息分类处理，那就会得到指定的消息。

首先要创建一个窗口。服务端逻辑：

> 网络库、头文件
> 打开网络库
> 校验版本
> 创建socket
> 绑定端口与IP
> 开始监听
> 事件选择：绑定事件与socket并且投递出去。

函数原型：

```c
int WSAAsyncSelect(SOCKET s, HWND hwnd, u_int wMsg, long IEvent);
```

参数1：要绑定的socket
参数2：要绑定的窗体
参数3：自定义消息（就是一个数），在回调函数可以捕获
参数4：要给服务器绑定的操作，FD_ACCEPT | FD_READ | FD_WRITE | FD_CLOSE

### 2. 例子

因为需要创建窗口，所以设置：项目属性->链接器->系统->子系统选择窗口

```c++
//#include <windows.h>
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <TCHAR.h>
#include <stdio.h>
#include <stdlib.h>
#include <Winsock2.h>
#include <string.h>

#pragma comment(lib, "Ws2_32.lib")

// WM_
#define UM_ASYNCSELECTMSG  WM_USER+1

LRESULT CALLBACK WinBackProc(HWND hWnd, UINT msgID, WPARAM wparam, LPARAM lparam);//声明回调函数

#define MAX_SOCK_COUNT 1024
SOCKET g_sockALL[MAX_SOCK_COUNT];	//存储有消息的socket，最后要释放
int g_count = 0;

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPreInstance, LPSTR lpCmdLine, int nShowCmd)
{
	//创建窗口结构体
	WNDCLASSEX wc;
	wc.cbClsExtra = 0;
	wc.cbSize = sizeof(WNDCLASSEX);
	wc.cbWndExtra = 0;
	wc.hbrBackground = NULL;
	wc.hCursor = NULL;
	wc.hIcon = NULL;
	wc.hIconSm = NULL;
	wc.hInstance = hInstance;
	wc.lpfnWndProc = WinBackProc;
	wc.lpszClassName = _T("c3window");//打开项目属性->常规->字符集, 如果使用多字节字符集则警告前加_T，如果使用Unicode字符集则警告前加L
	wc.lpszMenuName = NULL;
	wc.style = CS_HREDRAW | CS_VREDRAW;

	//注册结构体
	RegisterClassEx(&wc);

	//创建窗口
	HWND hWnd = CreateWindowEx(WS_EX_OVERLAPPEDWINDOW, TEXT("c3window"), _T("c3窗口"), WS_OVERLAPPEDWINDOW, 200, 200, 600, 400, NULL, NULL, hInstance, NULL);
	if (NULL == hWnd)
	{
		return 0;
	}

	//显示窗口
	ShowWindow(hWnd, nShowCmd);//1

	//更新窗口
	UpdateWindow(hWnd);

	WORD wdVersion = MAKEWORD(2, 2); //2.1  //22
	//int a = *((char*)&wdVersion);
	//int b = *((char*)&wdVersion+1);
	WSADATA wdScokMsg;
	//LPWSADATA lpw = malloc(sizeof(WSADATA));// WSADATA*
	int nRes = WSAStartup(wdVersion, &wdScokMsg);

	if (0 != nRes)
	{
		switch (nRes)
		{
		case WSASYSNOTREADY:
			printf("重启下电脑试试，或者检查网络库");
			break;
		case WSAVERNOTSUPPORTED:
			printf("请更新网络库");
			break;
		case WSAEINPROGRESS:
			printf("请重新启动");
			break;
		case WSAEPROCLIM:
			printf("请尝试关掉不必要的软件，以为当前网络运行提供充足资源");
			break;
		}
		return  0;
	}

	//校验版本
	if (2 != HIBYTE(wdScokMsg.wVersion) || 2 != LOBYTE(wdScokMsg.wVersion))
	{
		//说明版本不对
		//清理网络库
		WSACleanup();
		return 0;
	}

	SOCKET socketServer = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	//int a = WSAGetLastError();
	if (INVALID_SOCKET == socketServer)
	{
		int a = WSAGetLastError();
		//清理网络库
		WSACleanup();
		return 0;
	}

	struct sockaddr_in si;
	si.sin_family = AF_INET;
	si.sin_port = htons(12345);
	si.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");
	//int a = ~0;
	if (SOCKET_ERROR == bind(socketServer, (const struct sockaddr*)&si, sizeof(si)))
	{
		//出错了
		int a = WSAGetLastError();
		//释放
		closesocket(socketServer);
		//清理网络库
		WSACleanup();
		return 0;
	}

	if (SOCKET_ERROR == listen(socketServer, SOMAXCONN))
	{
		//出错了
		int a = WSAGetLastError();
		//释放
		closesocket(socketServer);
		//清理网络库
		WSACleanup();
		return 0;
	}
	//绑定窗体与socket并投递给操作系统
	if (SOCKET_ERROR == WSAAsyncSelect(socketServer, hWnd, UM_ASYNCSELECTMSG, FD_ACCEPT))
	{
		//出错了
		int a = WSAGetLastError();
		//释放
		closesocket(socketServer);
		//清理网络库
		WSACleanup();
		return 0;
	}

	g_sockALL[g_count] = socketServer;		//将刚创建的服务器socket存入socket数组
	g_count++;

	//消息循环
	MSG msg;
	while (GetMessage(&msg, NULL, 0, 0))
	{
		TranslateMessage(&msg);
		DispatchMessage(&msg);		//将接收到的消息传递给回调函数
	}

	//消息循环退出后，关闭所有socket
	for (int i = 0; i < g_count; i++)
	{
		closesocket(g_sockALL[i]);
	}
	//关闭网络库
	WSACleanup();
	return 0;
}

int x = 0;
//回调函数
LRESULT CALLBACK WinBackProc(HWND hWnd, UINT msgID, WPARAM wparam, LPARAM lparam)//将自定义消息（case判断的信息）通过参数2传递到回调函数中
																				 //将产生消息的客户端socket通过参数3传递到回调函数中
																				 //网络通信中产生的具体消息通过参数4传递进回调函数
{
	HDC hdc = GetDC(hWnd);//得到用户操作区域句柄
	switch (msgID)
	{
	case UM_ASYNCSELECTMSG:
	{
		//MessageBox(NULL, L"有信号啦", L"提示", MB_YESNO);
		//获取socket
		SOCKET sock = (SOCKET)wparam;
		//获取消息
		if (0 != HIWORD(lparam))//如果非正常消息，比如退出等
		{
			if (WSAECONNABORTED == HIWORD(lparam))//所有非正常消息均可通过WSAECONNABORTED捕捉（在此处捕捉到了退出消息）。
			{
				TextOut(hdc, 0, x, _T("close"), strlen("close"));
				x += 15;
				//关闭该socket上的消息
				WSAAsyncSelect(sock, hWnd, 0, 0);
				//关闭socket
				closesocket(sock);
				//记录数组中删除该socket
				for (int i = 0; i < g_count; i++)
				{
					if (sock == g_sockALL[i])
					{
						g_sockALL[i] = g_sockALL[g_count - 1];
						g_count--;
						break;
					}
				}
			}
			break;
		}
		//具体消息
		switch (LOWORD(lparam))
		{
		case FD_ACCEPT:
		{
			TextOut(hdc, 0, x, _T("accept"), strlen("accept"));
			x += 15;
			SOCKET socketClient = accept(sock, NULL, NULL);
			if (INVALID_SOCKET == socketClient)
			{
				//出错了
				int a = WSAGetLastError();
				break;
			}
			//将客户端投递给消息队列
			if (SOCKET_ERROR == WSAAsyncSelect(socketClient, hWnd, UM_ASYNCSELECTMSG, FD_READ | FD_WRITE | FD_CLOSE))
			{
				//出错了
				int a = WSAGetLastError();
				//释放
				closesocket(socketClient);
				break;
			}
			//记录
			g_sockALL[g_count] = socketClient;
			g_count++;
		}
		break;
		case FD_READ:
		{
			TextOut(hdc, 0, x, _T("read"), strlen("read"));
			char str[1024] = { 0 };
			if (SOCKET_ERROR == recv(sock, str, 1023, 0))
			{
				break;
			}
			// 自己添加的
			WCHAR wszClassName[1024];
			memset(wszClassName, 0, sizeof(wszClassName));
			MultiByteToWideChar(CP_ACP, 0, str, strlen(str) + 1, wszClassName,
				sizeof(wszClassName) / sizeof(wszClassName[0]));
			// 自己添加的
			TextOut(hdc, 30, x, wszClassName, wcslen(wszClassName));
			x += 15;
		}
		break;
		case FD_WRITE:
			//send 
			TextOut(hdc, 0, x, _T("wrtie"), strlen("wrtie"));
			x += 15;
			break;
		case FD_CLOSE:
			TextOut(hdc, 0, x, _T("close"), strlen("close"));
			x += 15;
			//关闭该socket上的消息
			WSAAsyncSelect(sock, hWnd, 0, 0);
			//关闭socket   
			closesocket(sock);
			//记录数组中删除该socket
			for (int i = 0; i < g_count; i++)
			{
				if (sock == g_sockALL[i])
				{
					g_sockALL[i] = g_sockALL[g_count - 1];
					g_count--;
					break;
				}
			}
		}
	}
	break;
	case WM_CREATE: //初始化 只执行一次

		break;
	case WM_DESTROY:
		PostQuitMessage(0);
		break;
	}

	ReleaseDC(hWnd, hdc);//释放用户区域句柄

	return DefWindowProc(hWnd, msgID, wparam, lparam);
}
```

## 4. 事件选择模型：WSAEventSelect

### 4.1 事件选择模型的意义

select模型克服了CS模型的几种缺点，但是select模型也存在缺陷，比如当select函数将所有socket投递给操作系统后，系统系统帮我们把有信号的socket装进fd_set结构体，直至返回有事件的socket集合这一过程程序是阻塞等待的，为了克服这个缺点人们提出了事件选择模型。

### 4.2 Windows处理用户行为的两种方式

#### 1. 事件机制

核心：事件集合
处理过程：根据需要，我们为用户的特定操作绑定一个事件，事件由我们自己调用API创建，需要多少就创建多少，将投递给系统，让系统帮忙监管。
特点：所有的事件都是用户自己定义的，系统只是帮用户置成有无信号，事件是没有顺序的（我们介绍的事件选择模型就是应用该机制）。

#### 2. 消息机制

核心：消息对列
处理过程：所有的用户操作，比如点击鼠标、键盘等，所有的操作均依次按顺序被记录，装进一个队列。
特点：消息对列由操作系统维护，咱们做的操作被操作系统取出来分类处理（有先后顺序）。

### 4.3 事件选择模型

#### 1. 事件选择模型的整体逻辑

创建一个事件对象（WSACreateEvent）；
为每一个事件对象绑定一个socket以及操作（accept、recv等）并投递给操作系统，程序员无需其它操作（注：这里select函数是循环检测是否有socket响应，而事件选择模型使主动件有响应的socket投递给操作系统）；
查看事件是否有信号（WSAWaitForMultipleEvents）；
有信号的话就分类处理（WSAEnumNetWorkEvents）；

#### 2. 函数详解

**创建一个事件对象**：`WSAEVENT eventSever = WSACreateEvent();`
参数：无
返回值：如果没有错误则返回事件对象的句柄；
    如果出错则返回 WSA_INVALID_EVENT。
**绑定并投递**：

```c
int WSAAPI WSAEventSelect(
  [in] SOCKET   s,
  [in] WSAEVENT hEventObject,
  [in] long     lNetworkEvents
);
```

功能：给事件绑上socket码与事件
参数1：被绑定的socket
参数2：事件对象，就是将参数1与参数2绑定在一起
参数3：具体事件（一般有四种：FD_ACCEPT、FD_READ、FD_WRITE和FD_CLOSE）。
返回值：如果成功返回0，如果失败则返回SOCKET_ERROR。

**询问事件**：

```c
DWORD WSAAPI WSAWaitForMultipleEvents(
  [in] DWORD          cEvents,
  [in] const WSAEVENT *lphEvents,
  [in] BOOL           fWaitAll,
  [in] DWORD          dwTimeout,
  [in] BOOL           fAlertable
);
```

功能：获取发生信号的事件。
参数1：事件个数
参数2：事件列表(我们定义一个结构体，存储所有的socket、事件，socket与事件一一对应，在结构体中定义一个变量存储socket与事件个数，参数1和参数2直接从该结构体取出即可)。
参数3：事情等待的方式，一般填TRUE或FALSE
  TRUE：所有的事件均产生信号才返回
  FALSE： （1）任何一个事件产生信号则立即返回；
         （2）返回值减去WSA_WAIT_EVENT_0表示事件对象的索引，其状态导致函数返回。
参数4：超时间隔，以毫秒为单位，跟select函数参数5含义一样
    n，等待n秒，超时返回WSA_WAIT_TIMEOUT
    0，检查事件对象的状态并立即返回（不管有没有信号）
    WSA_INFINTE，等待直到有事件发生
参数5：重叠IO模型中填TRUE，在事件选择模型中用FALSE即可。
返回值： 1. 我们要获得有信号事件发生的索引，若参数3为TRUE，则所有事件均有信号发生，无需特殊方式获得索引；如果参数3为FALSE，则返回值减去WSA_WAIT_EVENT_0就是数组中事件的下标。

     2. 如果返回值为WSA_WAIT_TIMEOUT，则代表超时了，continue即可。
     3如果返回值为WSA_WAIT_FAILED，则代表函数执行失败。

**列举事件**：

```c
typedef struct _WSANETWORKEVENTS {
       long lNetworkEvents;
       int iErrorCode[FD_MAX_EVENTS];
} WSANETWORKEVENTS, FAR * LPWSANETWORKEVENTS;

int WSAAPI WSAEnumNetworkEvents(
  [in]  SOCKET             s,
  [in]  WSAEVENT           hEventObject,
  [out] LPWSANETWORKEVENTS lpNetworkEvents
);
```

功能：获取事件类型（accept、recv、close等），并将事件上的信号量重置。
参数1：对应的socket；
参数2：对应的事件；
参数3：触发事件的类型在这里装着；它是一个结构体指针，系统已经为我们定义好了：
成员1（lNetworkEvents）：具体操作，一个信号可能包含两个操作，以按位或的形式存在。
成员2（iErrorCode）：错误码数组，FD_ACCEPT事件错误码存在该数组中，对应下标为FD_ACCEPT_BIT，如果没有对应就是0；
返回值：如果成功返回0；如果失败返回SOCKET_ERROR。

### 4.4 例子

说明：本节给出服务端的程序，客户端的程序和select模型中的客户端程序一样，在此不重复给出。运行时仍需要先开启服务端，再开启多个客户端。上一篇中的select模型和本篇中的事件选择模型对于我们普通用户感觉差不多，但是对于系统来说事件选择模型效率确实是提高了。

事件选择模型对于select模型来说明显提高了程序的执行效率，但是程序在执行到recv、send函数在向收发缓冲区读取或写入数据时仍然是阻塞的，要想从根本上克服这个缺点，利用多线程机制是一种行之有效的方法。

```c++
//服务端
#define _CRT_SECURE_NO_WARNINGS
#define _WINSOCK_DEPRECATED_NO_WARNINGS
#include <stdio.h>
#include <WinSock2.h>
#include <stdbool.h>
#pragma comment(lib,"ws2_32.lib")

struct fd_es_set {
	unsigned short count;
	SOCKET sockall[WSA_MAXIMUM_WAIT_EVENTS];//WSA_MAXIMUM_WAIT_EVENTS是64
	WSAEVENT eventall[WSA_MAXIMUM_WAIT_EVENTS];
};
struct fd_es_set esSet; // 全局变量不需要初始化，变量自动被置成0

BOOL WINAPI over(DWORD dwCtrlType)
{
	switch (dwCtrlType)
	{
	case CTRL_CLOSE_EVENT:
		//释放所有socket
		for (int i = 0; i < esSet.count; i++)
		{
			closesocket(esSet.sockall[i]);
			WSACloseEvent(esSet.eventall[i]);
		}
		break;
	}
	WSACleanup();
	return 0;
}

int main()
{
	SetConsoleCtrlHandler(over, TRUE);//这个函数的作用是当点击运行框右上角叉号关闭时，执行上面的over函数，系统调用函数

	WORD wdVersion = MAKEWORD(2, 2);		//使用网络库的版本
	WSADATA wdSockMsg;						//系统通过调用这个参数给我们一些配置信息
	int nRes = WSAStartup(wdVersion, &wdSockMsg);//打开网络库
	if (0 != nRes)	//如果打开网络库错误
	{
		switch (nRes)
		{
		case WSASYSNOTREADY:
			printf("可以重启电脑，或检查网络库");
			break;
		case WSAVERNOTSUPPORTED:
			printf("请更新网络库");
			break;
		case WSAEINPROGRESS:
			printf("Please reboot this software");
			break;
		case WSAEPROCLIM:
			printf("请关闭不必要的软件，以为当前网络提供充足资源");
			break;
		case WSAEFAULT:
			printf("参数错误");
			break;
		}
		return 0;
	}

	//版本校验
	if (2 != HIBYTE(wdSockMsg.wVersion) || 2 != LOBYTE(wdSockMsg.wVersion))
	{
		//如果版本错误
		WSACleanup();				//清理网络库
		return 0;
	}

	SOCKET socketSever = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);		//创建服务器socket
	if (INVALID_SOCKET == socketSever)
	{
		//socket创建失败
		int a = WSAGetLastError();	//返回错误码
		WSACleanup();				//清理网络库
		return 0;
	}

	struct sockaddr_in si;
	si.sin_family = AF_INET;
	si.sin_addr.S_un.S_addr = inet_addr("127.0.0.1");
	si.sin_port = htons(12345);
	int bind_a = bind(socketSever, (struct sockaddr*)&si, sizeof(si));
	if (SOCKET_ERROR == bind_a)
	{
		//如果bind函数出错
		int a = WSAGetLastError();	//返回错误码
		closesocket(socketSever);	//关闭socket
		WSACleanup();				//清理网络库
		return 0;
	}
	//开始监听
	int listen_r = listen(socketSever, SOMAXCONN);
	if (SOCKET_ERROR == listen_r)
	{
		//如果listen函数出错
		int a = WSAGetLastError();	//返回错误码
		closesocket(socketSever);	//关闭socket
		WSACleanup();				//清理网络库
		return 0;
	}

	//struct fd_es_set esSet = { 0,{0},{NULL} };

	//创建事件
	WSAEVENT eventSever = WSACreateEvent();
	if (WSA_INVALID_EVENT == eventSever)
	{
		//如果创建事件失败
		int a = WSAGetLastError();	//返回错误码
		closesocket(socketSever);	//关闭socket
		WSACleanup();				//清理网络库
		return 0;
	}
	//绑定并投递
	int wsa_r = WSAEventSelect(socketSever, eventSever, FD_ACCEPT);
	if (wsa_r == SOCKET_ERROR)
	{
		//如果绑定失败
		int a = WSAGetLastError();	//返回错误码
		//释放事件
		WSACloseEvent(eventSever);
		//释放所有socket
		closesocket(socketSever);
		//清理网络库
		WSACleanup();
		return 0;
	}
	//装进结构体
	esSet.eventall[esSet.count] = eventSever;
	esSet.sockall[esSet.count] = socketSever;
	esSet.count++;
	//printf("write event\n");
	while (1)
	{
		// 询问
		for (int nIndex = 0; nIndex < esSet.count; nIndex++)
		{//由于事件机制中的事件是无序的，这样可能会造成一些事件在某些情况下等待时间过长,因此在这里添加一个for循环使得事件有顺序
			DWORD nRes = WSAWaitForMultipleEvents(esSet.count, esSet.eventall, FALSE, 100, FALSE);
			if (WSA_WAIT_FAILED == nRes)
			{
				//如果是WSAWaitForMultipleEvents函数出错
				printf("错误码：%d\n", WSAGetLastError());
				break;
			}
			//如果WSAWaitForMultipleEvents参数4是一个具体的数，则需要使用下面的超时处理函数
			if (WSA_WAIT_TIMEOUT == nRes)
			{
				continue;
			}

			//DWORD nIndex = nRes - WSA_WAIT_EVENT_0;//返回值减WSA_WAIT_EVENT_0得到数组中事件的下标

			WSANETWORKEVENTS NetworkEvents;

			//得到下标后对应的具体操作
			if (SOCKET_ERROR == WSAEnumNetworkEvents(esSet.sockall[nIndex], esSet.eventall[nIndex], &NetworkEvents))
				//参数1：对应的socket，参数2：对应的事件，参数3：触发事件的类型
			{
				//如果WSAEnumNetworkEvents执行出错
				printf("错误码：%d\n", WSAGetLastError());
				break;
			}

			if (NetworkEvents.lNetworkEvents & FD_ACCEPT)
			{
				if (0 == NetworkEvents.iErrorCode[FD_ACCEPT_BIT])
				{
					//正常处理
					SOCKET socketClient = accept(socketSever, NULL, NULL);//socketSever也可替换为esSet.sockall[nIndex]
					if (INVALID_SOCKET == socketClient)
					{
						continue;
					}
					//创建事件对象
					WSAEVENT wsaClientEvent = WSACreateEvent();
					if (WSA_INVALID_EVENT == wsaClientEvent)
					{
						//事件对象创建失败
						closesocket(socketClient);
						continue;
					}
					//将刚创建的事件与socket绑定后投递给系统
					if (SOCKET_ERROR == WSAEventSelect(socketClient, wsaClientEvent, FD_READ | FD_WRITE | FD_CLOSE))
					{
						//如果绑定失败
						int a = WSAGetLastError();	//返回错误码
						//释放事件
						WSACloseEvent(eventSever);
						//释放所有socket
						closesocket(socketSever);
						continue;
					}
					//将刚accept返回的客户端socket与刚创建的事件装进结构体
					esSet.sockall[esSet.count] = socketClient;
					esSet.eventall[esSet.count] = wsaClientEvent;
					esSet.count++;
					printf("accept event\n");
				}
				else
				{
					continue;
				}
			}
			if (NetworkEvents.lNetworkEvents & FD_WRITE)
			{
				if (0 == NetworkEvents.iErrorCode[FD_WRITE_BIT])
				{
					if (SOCKET_ERROR == send(esSet.sockall[nIndex], "connect success", strlen("connect success"), 0))
					{
						printf("connect fail, error code:%d\n", WSAGetLastError());
						continue;
					}
					printf("write event\n");
				}
				else
				{
					printf("socket error code:%d\n", NetworkEvents.iErrorCode[FD_WRITE_BIT]);
					continue;
				}
			}
			if (NetworkEvents.lNetworkEvents & FD_READ)
			{
				if (0 == NetworkEvents.iErrorCode[FD_READ_BIT])
				{
					char str[1500] = { 0 };
					if (SOCKET_ERROR == recv(esSet.sockall[nIndex], str, 1499, 0))
					{
						printf("recv error code:%d\n", WSAGetLastError());
						continue;
					}
					printf("%s\n", str);
				}
				else
				{
					printf("socket error code:%d\n", NetworkEvents.iErrorCode[FD_READ_BIT]);
					continue;
				}
			}
			if (NetworkEvents.lNetworkEvents & FD_CLOSE)
			{
				if (0 == NetworkEvents.iErrorCode[FD_CLOSE_BIT])
				{

				}
				else
				{
					printf("Client offline:%d\n", NetworkEvents.iErrorCode[FD_CLOSE_BIT]);
				}
				//如果某个socket出现错误，则需要将该socket对应的事件连同出错的socket本身一并从结构体中删除，并释放相关的资源
				//套接字
				closesocket(esSet.sockall[nIndex]);
				esSet.sockall[nIndex] = esSet.sockall[esSet.count - 1];
				//事件
				WSACloseEvent(esSet.eventall[nIndex]);
				esSet.eventall[nIndex] = esSet.eventall[esSet.count - 1];
				esSet.count--;
			}
		}
	}
	//释放事件数组和socket数组
	for (int i = 0; i < esSet.count; i++)
	{
		closesocket(esSet.sockall[i]);
		WSACloseEvent(esSet.eventall[i]);
	}
	//清理网络库
	WSACleanup();
	return 0;
}
```

## 4. 重叠模型

### 4.1 重叠模型的优点

1. 可以运行在支持Winsock2的所有Windows平台 ,而不像完成端口只是支持NT系统。

2. 比起阻塞、select、WSAAsyncSelect以及WSAEventSelect等模型，重叠I/O(Overlapped I/O)模型使应用程序能达到更佳的系统性能。

   因为它和这4种模型不同的是，使用重叠模型的应用程序通知缓冲区收发系统直接使用数据，也就是说，如果应用程序投递了一个10KB大小的缓冲区来接收数据，且数据已经到达套接字，则该数据将直接被拷贝到投递的缓冲区。

   而这4种模型种，数据到达并拷贝到单套接字接收缓冲区中，此时应用程序会被告知可以读入的容量。当应用程序调用接收函数之后，数据才从单套接字缓冲区拷贝到应用程序的缓冲区，差别就体现出来了。

3.      从《windows网络编程》中提供的试验结果中可以看到，在使用了P4 1.7G Xero处理器(CPU很强啊)以及768MB的回应服务器中，最大可以处理4万多个SOCKET连接，在处理1万2千个连接的时候CPU占用率才40% 左右 ―― 非常好的性能，已经直逼完成端口了^_^

### 4.2 重叠模型的基本原理

概括一点说，重叠模型是让应用程序使用重叠数据结构(WSAOVERLAPPED)，一次投递一个或多个Winsock I/O请求。针对这些提交的请求，在它们完成之后，应用程序会收到通知，于是就可以通过自己另外的代码来处理这些数据了。
需要注意的是，有两个方法可以用来管理重叠IO请求的完成情况（就是说接到重叠操作完成的通知）：
1.      事件对象通知(event object notification)
2.      完成例程(completion routines) ,注意，这里并不是完成端口

而本文只是讲述如何来使用事件通知的的方法实现重叠IO模型。既然是基于事件通知，就要求将Windows事件对象与WSAOVERLAPPED结构关联在一起（WSAOVERLAPPED结构中专门有对应的参数）。既然要使用重叠结构，我们常用的send, sendto, recv, recvfrom也都要被WSASend, WSASendto, WSARecv, WSARecvFrom替换掉了， 它们的用法我后面会讲到，这里只需要注意一点，它们的参数中都有一个Overlapped参数，我们可以假设是把我们的WSARecv这样的操作操作“绑定”到这个重叠结构上，提交一个请求，其他的事情就交给重叠结构去操心，而其中重叠结构又要与Windows的事件对象“绑定”在一起，这样我们调用完WSARecv以后就可以“坐享其成”，等到重叠操作完成以后，自然会有与之对应的事件来通知我们操作完成，然后我们就可以来根据重叠操作的结果取得我们想要德数据了。

### 4.3 关于重叠模型的基础知识

下面来介绍并举例说明一下编写重叠模型的程序中将会使用到的几个关键函数。

#### 1. WSAOVERLAPPED结构

这个结构自然是重叠模型里的核心，它是这么定义的

```c
typedef struct _WSAOVERLAPPED {
  DWORD Internal;
  DWORD InternalHigh;
  DWORD Offset;
  DWORD OffsetHigh;
  WSAEVENT hEvent;      // 唯一需要关注的参数，用来关联WSAEvent对象
} WSAOVERLAPPED, *LPWSAOVERLAPPED;
```

我们需要把WSARecv等操作投递到一个重叠结构上，而我们又需要一个与重叠结构“绑定”在一起的事件对象来通知我们操作的完成，看到了和hEvent参数，不用我说你们也该知道如何来来把事件对象绑定到重叠结构上吧？大致如下：

```c
WSAEVENT event;                   // 定义事件
WSAOVERLAPPED AcceptOverlapped ; // 定义重叠结构
event = WSACreateEvent();         // 建立一个事件对象句柄
ZeroMemory(&AcceptOverlapped, sizeof(WSAOVERLAPPED)); // 初始化重叠结构
AcceptOverlapped.hEvent = event;    // Done !！
```

#### 2. *WSARecv*系列函数

在重叠模型中，接收数据就要靠它了，它的参数也比*recv*要多，因为要用到重叠结构嘛，它是这样定义的：

```c
int WSARecv(
    SOCKET s,                      // 当然是投递这个操作的套接字
    LPWSABUF lpBuffers,          // 接收缓冲区，与Recv函数不同，这里需要一个由WSABUF结构构成的数组
    DWORD dwBufferCount,        // 数组中WSABUF结构的数量
    LPDWORD lpNumberOfBytesRecvd,  // 如果接收操作立即完成，这里会返回函数调用所接收到的字节数
    LPDWORD lpFlags,             // 说来话长了，我们这里设置为0 即可
    LPWSAOVERLAPPED lpOverlapped,  // “绑定”的重叠结构
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine); // 完成例程中将会用到的参数，我们这里设置为 NULL
```

返回值：
WSA_IO_PENDING ： 最常见的返回值，这是说明我们的WSARecv操作成功了，但是
                    I/O操作还没有完成，所以我们就需要绑定一个事件来通知我们操作何时完成

举个例子：(变量的定义顺序和上面的说明的顺序是对应的，下同)

```c
SOCKET s;
WSABUF DataBuf;           // 定义WSABUF结构的缓冲区
// 初始化一下DataBuf
#define DATA_BUFSIZE 5096
char buffer[DATA_BUFSIZE];
ZeroMemory(buffer, DATA_BUFSIZE);
DataBuf.len = DATA_BUFSIZE;
DataBuf.buf = buffer;
DWORD dwBufferCount = 1, dwRecvBytes = 0, Flags = 0;
// 建立需要的重叠结构
WSAOVERLAPPED AcceptOverlapped ;// 如果要处理多个操作，这里当然需要一个
// WSAOVERLAPPED数组
WSAEVENT event;     // 如果要多个事件，这里当然也需要一个WSAEVENT数组
                    // 需要注意的是可能一个SOCKET同时会有一个以上的重叠请求，也就会对应一个以上的WSAEVENT
Event = WSACreateEvent();
ZeroMemory(&AcceptOverlapped, sizeof(WSAOVERLAPPED));
AcceptOverlapped.hEvent = event;     // 关键的一步，把事件句柄“绑定”到重叠结构上
// 作了这么多工作，终于可以使用WSARecv来把我们的请求投递到重叠结构上了，呼。。。。
WSARecv(s, &DataBuf, dwBufferCount, &dwRecvBytes, &Flags, &AcceptOverlapped, NULL);
```

#### 3. WSAWaitForMultipleEvents函数

熟悉WSAEventSelect模型的朋友对这个函数肯定不会陌生，不对，其实大家都不应该陌生，这个函数与线程中常用的WaitForMultipleObjects函数有些地方还是比较像的，因为都是在等待某个事件的触发嘛。

因为我们需要事件来通知我们重叠操作的完成，所以自然需要这个等待事件的函数与之配套。

```c
DWORD WSAWaitForMultipleEvents(
    DWORD cEvents,                        // 等候事件的总数量
    const WSAEVENT* lphEvents,           // 事件数组的指针
    BOOL fWaitAll,          // 这个要多说两句：如果设置为 TRUE，则事件数组中所有事件被传信的时候函数才会返回
                            // FALSE则任何一个事件被传信函数都要返回，我们这里肯定是要设置为FALSE的
    DWORD dwTimeout,        // 超时时间
    BOOL fAlertable);       // 在完成例程中会用到这个参数，这里我们先设置为FALSE
```
返回值：
    WSA_WAIT_TIMEOUT ：最常见的返回值，我们需要做的就是继续Wait
    WSA_WAIT_FAILED ： 出现了错误，请检查cEvents和lphEvents两个参数是否有效
如果事件数组中有某一个事件被传信了，函数会返回这个事件的索引值，但是这个索引值需要减去预定义值 WSA_WAIT_EVENT_0才是这个事件在事件数组中的位置。
注意：WSAWaitForMultipleEvents函数只能支持由WSA_MAXIMUM_WAIT_EVENTS对象定义的一个最大值，是 64，就是说WSAWaitForMultipleEvents只能等待64个事件，如果想同时等待多于64个事件，就要 创建额外的工作者线程，就不得不去管理一个线程池，这一点就不如下一篇要讲到的完成例程模型了。

#### 4. WSAGetOverlappedResult函数

既然我们可以通过WSAWaitForMultipleEvents函数来得到重叠操作完成的通知，那么我们自然也需要一个函数来查询一下重叠操作的结果，定义如下

    BOOL WSAGetOverlappedResult(
        SOCKET s,                   // SOCKET，不用说了
        LPWSAOVERLAPPED lpOverlapped,  // 这里是我们想要查询结果的那个重叠结构的指针
        LPDWORD lpcbTransfer,     // 本次重叠操作的实际接收(或发送)的字节数
        BOOL fWait,                // 设置为TRUE，除非重叠操作完成，否则函数不会返回
                                   // 设置FALSE，而且操作仍处于挂起状态，那么函数就会返回FALSE
                                  // 错误为WSA_IO_INCOMPLETE
        LPDWORD lpdwFlags);       // 指向DWORD的指针，负责接收结果标志
这个函数没什么难的，这里我们也不需要去关注它的返回值，直接把参数填好调用就可以了，这里就先不举例了
唯一需要注意一下的就是如果WSAGetOverlappedResult完成以后，第三个参数返回是 0 ，则说明通信对方已经关闭连接，我们这边的SOCKET, Event之类的也就可以关闭了。

### 4.4 多客户端情况的注意事项

完成了上面的循环以后，重叠模型就已经基本上搭建好了80%了，为什么不是100%呢？因为仔细一回想起来，呀~~~~~~~，这样岂不是只能连接一个客户端？？是的，如果只处理一个客户端，那重叠模型就半点优势也没有了，我们正是要使用重叠模型来处理多个客户端。所以我们不得不再对结构作一些改动。

1． 首先，肯定是需要一个SOCKET数组 ,分别用来和每一个SOCKET通信

其次，因为重叠模型中每一个SOCKET操作都是要“绑定”一个重叠结构的，所以需要为每一个SOCKET操作搭配一个WSAOVERLAPPED结构，但是这样说并不严格，因为如果每一个SOCKET同时只有一个操作，比如WSARecv，那么一个SOCKET就可以对应一个WSAOVERLAPPED结构，但是如果一个SOCKET上会有WSARecv 和WSASend两个操作，那么一个SOCKET肯定就要对应两个WSAOVERLAPPED结构，所以有多少个SOCKET操作就会有多少个WSAOVERLAPPED结构。

然后，同样是为每一个WSAOVERLAPPED结构都要搭配一个WSAEVENT事件，所以说有多少个SOCKET操作就应该有多少个WSAOVERLAPPED结构，有多少个WSAOVERLAPPED结构就应该有多少个WSAEVENT事件，最好把SOCKET – WSAOVERLAPPED – WSAEVENT三者的关联起来，到了关键时刻才会临危不乱：）

2． 不得不分作两个线程：

一个用来循环监听端口，接收请求的连接，然后给在这个套接字上配合一个WSAOVERLAPPED结构投递第一个WSARecv请求，然后进入第二个线程中等待操作完成。

第二个线程用来不停的对WSAEVENT数组WSAWaitForMultipleEvents，等待任何一个重叠操作的完成，然后根据返回的索引值进行处理，处理完毕以后再继续投递另一个WSARecv请求。

这里需要注意一点的是，前面我是把WSAWaitForMultipleEvents函数的参数设置为WSA_INFINITE的，但是在多客户端的时候这样就不OK了，需要设定一个超时时间，如果等待超时了再重新WSAWaitForMultipleEvents，因为WSAWaitForMultipleEvents函数在没有触发的时候是阻塞在那里的，我们可以设想一下，这时如果监听线程忠接入了新的连接，自然也会为这个连接增加一个Event，但是WSAWaitForMultipleEvents还是阻塞在那里就不会处理这个新连接的Event了。

### 4.5 例子

ServerSocket.h

```c++
#pragma once

#include <WinSock2.h>
#pragma comment (lib,"ws2_32.lib")

class CServerSocket
{
	friend UINT _ServerListenThread(LPVOID lParam);
	friend UINT _OverlappedThread(LPVOID lParam);

public:
	CServerSocket(void);
	~CServerSocket(void);

	int StartListening(const UINT&);
	int StopListening();

private:
	int GetEmptySocket();

	UINT m_nPort;
};
```

ServerSocket.cpp

```c++
#include "serversocket.h"

#define DATA_BUFSIZE 4096

CWinThread* pServerListenThread = NULL;
CWinThread* pOverlappedThread = NULL;

SOCKET sockListen;

SOCKET sockAcceptArray[WSA_MAXIMUM_WAIT_EVENTS]; // 与客户端通信的SOCKET

WSAEVENT EventArray[WSA_MAXIMUM_WAIT_EVENTS];    // 与重叠结构配套的事件组

WSAOVERLAPPED AcceptOverlapped[WSA_MAXIMUM_WAIT_EVENTS]; // 重叠结构，每个SOCKET操作对应一个

WSABUF DataBuf[WSA_MAXIMUM_WAIT_EVENTS];  // 接收缓冲区，WSARecv的参数，每个SCOKET对应一个

DWORD dwEventTotal = 0, 
      dwRecvBytes = 0,
	  Flags = 0;

int  nSockIndex = 0;            // socket数组的编号

BOOL bOverlapped = FALSE;       // 是否处理重叠请求

// 监听线程
UINT _ServerListenThread(LPVOID lParam)
{
	TRACE("服务器端监听中.....\n");

	CServerSocket* pServer = (CServerSocket*)lParam;

	SOCKADDR_IN ClientAddr;                   // 定义一个客户端得地址结构作为参数
	int addr_length=sizeof(ClientAddr);

	while(TRUE)
	{
		if(dwEventTotal>=WSA_MAXIMUM_WAIT_EVENTS)  // 因为超出了Windows的最大等待事件数量
		{
			AfxMessageBox("已达到最大连接数！");
			continue;
		}

		SOCKET sockTemp = accept(sockListen,(SOCKADDR*)&ClientAddr,&addr_length); 
		if(sockTemp  == INVALID_SOCKET)
		{
			AfxMessageBox("Accept Connection failed！");
			continue;
		}

		nSockIndex = pServer->GetEmptySocket();    // 从Socket数组中获得一个空闲的socket

		sockAcceptArray[nSockIndex] = sockTemp;
		
		// 这里可以取得客户端的IP和端口，但是我们只取其中的SOCKET编号
		LPCTSTR lpIP =  inet_ntoa(ClientAddr.sin_addr);
		UINT nPort = ClientAddr.sin_port;
		CString strSock;
		strSock.Format("SOCKET编号:%d",sockAcceptArray[nSockIndex] );
		::SendMessage(pServer->m_hNotifyWnd,WM_MSG_NEW_SOCKET,
			(LPARAM)(LPCTSTR)strSock,(LPARAM)(LPCTSTR)"客户端建立连接！");


		// 接收客户端连接以后，为每一个连入的SOCKET都初始化建立一个重叠结构
		Flags = 0;

		EventArray[nSockIndex] = WSACreateEvent();

		ZeroMemory(&AcceptOverlapped[nSockIndex],sizeof(WSAOVERLAPPED));

		char buffer[DATA_BUFSIZE]; // 局部变量离开范围会释放
		ZeroMemory(buffer,DATA_BUFSIZE);

		AcceptOverlapped[nSockIndex].hEvent = EventArray[nSockIndex]; // 关联事件

		DataBuf[nSockIndex].len = DATA_BUFSIZE;
		DataBuf[nSockIndex].buf = buffer; // buffer释放后变成野指针


	
		// 投递第一个WSARecv请求，以便开始在套接字上接受数据
		if(WSARecv(sockAcceptArray[nSockIndex] ,&DataBuf[nSockIndex],1,&dwRecvBytes,&Flags,
			&AcceptOverlapped[nSockIndex] ,NULL) == SOCKET_ERROR)
		{
			if(WSAGetLastError() != WSA_IO_PENDING)    
			{
				// 返回WSA_IO_PENDING是正常情况，表示IO操作正在进行，不能立即完成
				// 如果不是WSA_IO_PENDING错误，就表示操作失败了
				AfxMessageBox("错误：第一次投递Recv操作失败！！此套接字将被关闭!");
				closesocket(sockAcceptArray[nSockIndex]);
				sockAcceptArray[nSockIndex] = INVALID_SOCKET;

				WSACloseEvent(EventArray[nSockIndex]);

				continue;
			}
		}

		dwEventTotal++;

		if(dwEventTotal == 1)                          // 此时如果dwEventTotal加以后等于1 
			                                           // 就说明_OverlappedThread休眠了，此时唤醒之
			pOverlappedThread->ResumeThread();       
	}

	return 0;
}

// 重叠I/O处理线程
UINT _OverlappedThread(LPVOID lParam)
{
	CServerSocket* pServer = (CServerSocket*)lParam;

	while(bOverlapped)                    // 循环检测事件数组中的事件，并对接收的数据进行处理:)
	{
		DWORD dwIndex;

		// 等候重叠I/O调用结束
		// 因为我们把事件和Overlapped绑定在一起，重叠操作完成后我们会接到事件通知
		dwIndex = WSAWaitForMultipleEvents(dwEventTotal,EventArray,FALSE,10,FALSE);

		if(dwIndex == WSA_WAIT_TIMEOUT)
			continue;

		if(dwIndex == WSA_WAIT_FAILED)  // 出现监听错误
		{
			int nErrorCode = WSAGetLastError();

			if(nErrorCode == WSA_INVALID_HANDLE)
			{
				AfxMessageBox("监听出现错误：无效的 lphEvents 参数！");   // 代码经常会出现这种错误，我实在没时间改了，55555555
			}
			else if(nErrorCode == WSA_INVALID_PARAMETER)
			{
				AfxMessageBox("监听出现错误：无效的 CEvents 参数！");
			}

			continue;
		}

		// 取得索引值，得知事件的索引号
		dwIndex = dwIndex - WSA_WAIT_EVENT_0;

		// 获得索引号以后，这个事件已经没有利用价值，重置之
		WSAResetEvent(EventArray[dwIndex]);

		// 然后确定当前索引号的SOCKET的重叠请求状态
		DWORD dwBytesTransferred;
		WSAOVERLAPPED& CurrentOverlapped = AcceptOverlapped[dwIndex]; // 这里纯粹为了减少代码长度，付给了一个临时变量
		SOCKET& sockCurrent = sockAcceptArray[dwIndex] ;  // 同上

		WSAGetOverlappedResult(sockCurrent,&CurrentOverlapped ,
			&dwBytesTransferred,FALSE,&Flags);

		// 先检查通信对方是否已经关闭连接
		// 如果==0则表示连接已经，则关闭套接字
		if(dwBytesTransferred == 0)
		{
			CString strSock;
			strSock.Format("SOCKET编号:%d",sockAcceptArray[dwIndex] );
			::SendMessage(pServer->m_hNotifyWnd,WM_MSG_NEW_SOCKET,
				(LPARAM)(LPCTSTR)strSock,(LPARAM)(LPCTSTR)"客户端断开连接！");

			closesocket(sockCurrent);
			sockCurrent = INVALID_SOCKET;
			//WSACloseEvent( EventArray[dwIndex] );

			dwEventTotal--;
			if(dwEventTotal <= 0)                  // 如果没有事件等待则暂停重叠IO处理线程
			{

				pOverlappedThread->SuspendThread();  // 没有需要处理的重叠结构了，则本线程休眠
		    }
			continue;
		}


		// DataBuf中包含接收到的数据，我们发到对话框中给予显示
		CString strSock;
		strSock.Format("SOCKET编号:%d",sockCurrent);
		::SendMessage(pServer->m_hNotifyWnd,WM_MSG_NEW_MSG,
			(LPARAM)(LPCTSTR)strSock,(LPARAM)(LPCTSTR)DataBuf[dwIndex].buf);


		// 然后在套接字上投递另一个WSARecv请求(不要晕，注意看，代码和前面第一次WSARecv一样^_^)
		Flags = 0;
		ZeroMemory(&CurrentOverlapped,sizeof(WSAOVERLAPPED));

		char buffer[DATA_BUFSIZE];
		ZeroMemory(buffer,DATA_BUFSIZE);

		CurrentOverlapped.hEvent = EventArray[dwIndex];
		DataBuf[dwIndex].len = DATA_BUFSIZE;
		DataBuf[dwIndex].buf = buffer;

		// 开始另外一个WSARecv
		if(WSARecv(sockCurrent,&DataBuf[dwIndex],1,&dwRecvBytes,&Flags,
			&CurrentOverlapped ,NULL) == SOCKET_ERROR)
		{
			if(WSAGetLastError() != WSA_IO_PENDING) 
			{
				AfxMessageBox("错误：投递Recv操作失败！！此套接字将被关闭!");

				closesocket(sockCurrent);
				sockCurrent = INVALID_SOCKET;

				WSACloseEvent(EventArray[dwIndex]);

				dwEventTotal--;

				continue;
			}
		}
	}

	return 0;
}

CServerSocket::CServerSocket(void)
{
	for(int i=0;i<WSA_MAXIMUM_WAIT_EVENTS;i++)   // 初始化SOCKET
	{
		sockAcceptArray[i] = INVALID_SOCKET;
	}
}

CServerSocket::~CServerSocket(void)
{
	TRACE("CServerSocket 析构！");

	if(pServerListenThread != NULL)
	{
		DWORD   dwStatus;
		::GetExitCodeThread(pServerListenThread->m_hThread, &dwStatus);

		if (dwStatus == STILL_ACTIVE)
		{
			delete pServerListenThread;
			pServerListenThread = NULL;
		}
	}

	if(pOverlappedThread != NULL)
	{
		DWORD   dwStatus;
		::GetExitCodeThread(pOverlappedThread->m_hThread, &dwStatus);

		if (dwStatus == STILL_ACTIVE)
		{
			delete pOverlappedThread;
			pOverlappedThread = NULL;
		}
	}
	
	WSACleanup();
}

// 这些都是固定套路了，没有区别
int CServerSocket::StartListening(const UINT& port)
{
	m_nPort = port;

	WSADATA wsaData;

	int nRet;

	nRet=WSAStartup(MAKEWORD(2,2),&wsaData);                      //开启winsock.dll
	if(nRet!=0)
	{
		AfxMessageBox("Load winsock2 failed");
		WSACleanup();
		return -1;
	}

	sockListen = socket(AF_INET,SOCK_STREAM,IPPROTO_TCP);         //创建服务套接字(流式)

	SOCKADDR_IN ServerAddr;                                       //分配端口及协议族并绑定

	ServerAddr.sin_family=AF_INET;                                
	ServerAddr.sin_addr.S_un.S_addr  =htonl(INADDR_ANY);          
	ServerAddr.sin_port=htons(port);

	nRet=bind(sockListen,(LPSOCKADDR)&ServerAddr,sizeof(ServerAddr)); // 绑定套接字

	if(nRet==SOCKET_ERROR)
	{
		AfxMessageBox("Bind Socket Fail!");
		closesocket(sockListen);
		return -1;
	}

	nRet=listen(sockListen,5);                                   //开始监听，并设置监听客户端数量
	if(nRet==SOCKET_ERROR)     
	{
		AfxMessageBox("Listening Fail!");
		return -1;
	}


	pServerListenThread =  AfxBeginThread(_ServerListenThread,this);    // 开始监听线程

	bOverlapped = TRUE;

	// 开始重叠IO线程，但是先休眠，因为没有需要等待的事件。
	pOverlappedThread = AfxBeginThread(_OverlappedThread,this,THREAD_PRIORITY_NORMAL,0,CREATE_SUSPENDED,NULL); 

	return 0;
}

int CServerSocket::StopListening()
{
	bOverlapped = FALSE;

	return 0;
}

// 返回一个空闲的SOCKET，算法可以改进的，这样穷举效率太低
int CServerSocket::GetEmptySocket()
{
	for(int i=0;i<WSA_MAXIMUM_WAIT_EVENTS;i++)
	{
		if(sockAcceptArray[i]==INVALID_SOCKET)
		{
			return i;
		}
	}

	return -1;
}
```

## 5. 完成例程(Completion Routine)

### 5.1 完成例程的优点

1. 首先需要指明的是，这里的“完成例程”(Completion Routine)并非是大家所常听到的“完成端口”(Completion Port)，而是另外一种管理重叠I/O请求的方式，而至于什么是重叠I/O，简单来讲就是Windows系统内部管理I/O的一种方式，核心就是调用的ReadFile和WriteFile函数，在制定设备上执行I/O操作，不光是可用于网络通信，也可以用于其他需要的地方。

在Windows系统中，管理重叠I/O可以有三种方式：

(1) 上一篇中提到的基于事件通知的重叠I/O模型

 (2) 本篇中将要讲述的基于“完成例程”的重叠I/O模型

 (3) 下一篇中将要讲到的“完成端口”模型

虽然都是基于重叠I/O，但是因为前两种模型都是需要自己来管理任务的分派，所以性能上没有区别，而完成端口是创建完成端口对象使操作系统亲自来管理任务的分派，所以完成端口肯定是能获得最好的性能。

2.   如果你想要使用重叠I/O机制带来的高性能模型，又懊恼于基于事件通知的重叠模型要收到64个等待事件的限制，还有点畏惧完成端口稍显复杂的初始化过程，那么“完成例程”无疑是你最好的选择！^_^因为完成例程摆脱了事件通知的限制，可以连入任意数量客户端而不用另开线程，也就是说只用很简单的一些代码就可以利用Windows内部的I/O机制来获得网络服务器的高性能。

3.   而且个人感觉“完成例程”的方式比重叠I/O更好理解，因为就和我们传统的“回调函数”是一样的，也更容易使用一些，推荐！

### 5.2 完成例程的基本原理

概括一点说，上一篇拙作中提到的那个基于事件通知的重叠I/O模型，在你投递了一个请求以后(比如WSARecv)，系统在完成以后是用事件来通知你的，而在完成例程中，系统在网络操作完成以后会自动调用你提供的回调函数，区别仅此而已，是不是很简单呢？

首先这里统一几个名词，包括“重叠操作”、“重叠请求”、“投递请求”等等，这是为了配合这的重叠I/O才这么讲的，说的直白一些，也就是你在代码中发出的WSARecv()、WSASend()等等网络函数调用。

 上篇文章中偷懒没画图，这次还是画个流程图来说明吧，采用完成例程的服务器端，通信流程简单的来讲是这样的：

![](.\pic\CompletionRoutine说明.jpg)

从图中可以看到，服务器端存在一个明显的异步过程，也就是说我们把客户端连入的SOCKET与一个重叠结构绑定之后，便可以将通讯过程全权交给系统内部自己去帮我们调度处理了，我们在主线程中就可以去做其他的事情，边等候系统完成的通知就OK，这也就是完成例程高性能的原因所在。

如果还没有看明白，我们打个通俗易懂的比方，完成例程的处理过程，也就像我们告诉系统，说“我想要在网络上接收网络数据，你去帮我办一下”（投递WSARecv操作），“不过我并不知道网络数据合适到达，总之在接收到网络数据之后，你直接就调用我给你的这个函数(比如_CompletionProess)，把他们保存到内存中或是显示到界面中等等，全权交给你处理了”，于是乎，系统在接收到网络数据之后，一方面系统会给我们一个通知，另外同时系统也会自动调用我们事先准备好的回调函数，就不需要我们自己操心了。

### 5.3 完成例程的函数介绍

完成例程方式和前面的事件通知方式最大的不同之处就在于，我们需要提供一个回调函数供系统收到网络数据后自动调用，回调函数的参数定义应该遵照如下的函数原型：

```c
Void CALLBACK _CompletionRoutineFunc(
  DWORD dwError, // 标志咱们投递的重叠操作，比如WSARecv，完成的状态是什么
  DWORD cbTransferred, // 指明了在重叠操作期间，实际传输的字节量是多大
  LPWSAOVERLAPPED lpOverlapped, // 参数指明传递到最初的IO调用内的一个重叠结构
  DWORD dwFlags); // 返回操作结束时可能用的标志(一般没用)
```

 还有一点需要重点提一下的是，因为我们需要给系统提供一个如上面定义的那样的回调函数，以便系统在完成了网络操作后自动调用，这里就需要提一下究竟是如何把这个函数与系统内部绑定的呢？如下所示，在WSARecv函数中是这样绑定的：

```c
int WSARecv(
    SOCKET s,                      // 当然是投递这个操作的套接字
    LPWSABUF lpBuffers,          // 接收缓冲区，与Recv函数不同，这里需要一个由WSABUF结构构成的数组
    DWORD dwBufferCount,        // 数组中WSABUF结构的数量，设置为1即可
    LPDWORD lpNumberOfBytesRecvd,  // 如果接收操作立即完成，这里会返回函数调用所接收到的字节数
    LPDWORD lpFlags,             // 说来话长了，我们这里设置为0 即可
    LPWSAOVERLAPPED lpOverlapped,  // “绑定”的重叠结构
    LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine); // 我们的完成例程函数的指针
```

举个例子：(变量的定义顺序和上面的说明的顺序是对应的，下同)

```c
SOCKET s;
WSABUF DataBuf;           // 定义WSABUF结构的缓冲区
// 初始化一下DataBuf
#define DATA_BUFSIZE 4096
char buffer[DATA_BUFSIZE];
ZeroMemory(buffer, DATA_BUFSIZE);
DataBuf.len = DATA_BUFSIZE;
DataBuf.buf = buffer;
DWORD dwBufferCount = 1, dwRecvBytes = 0, Flags = 0;
// 建立需要的重叠结构，每个连入的SOCKET上的每一个重叠操作都得绑定一个
WSAOVERLAPPED AcceptOverlapped ;// 如果要处理多个操作，这里当然需要一个WSAOVERLAPPED数组
ZeroMemory(&AcceptOverlapped, sizeof(WSAOVERLAPPED));

// 作了这么多工作，终于可以使用WSARecv来把我们的完成例程函数绑定上了
// 当然，假设我们的_CompletionRoutine函数已经定义好了
WSARecv(s, &DataBuf, dwBufferCount, &dwRecvBytes, 
&Flags, &AcceptOverlapped, _CompletionRoutine);
```

等待事件完成也可以使用SleepEx：

```c++
DWORD SleepEx(
    DWORD dwMilliseconds,  // 等待的超时时间，如果设置为INFINITE就会一直等待下去
    BOOL   bAlertable);   // 是否置于警觉状态，如果为FALSE，则一定要等待超时时间完毕之后才会返回，这里我们是希望重叠操作一完成就能返回，所以同(1)一样，我们一定要设置为TRUE
```

### 5.4 例子

其实就是把用事件通知的方式改成了回调。

ServerSocket.h

```c++
#pragma once

// 包含Winsock2必须的头文件和库文件
#include <WinSock2.h>
#pragma comment (lib,"ws2_32.lib")

class CServerSocket
{
	friend UINT _ServerListenThread(LPVOID lParam);

public:

	CServerSocket(void);
	~CServerSocket(void);

	int StartListening(const UINT&);
	int StopListening();

	static void ShowMessage(const int, const CString&);
	static HWND m_hNotifyWnd;

private:
	int GetEmptySocket();

	UINT m_nPort;
};
```

ServerSocket.cpp

```c++
#include "StdAfx.h"
#include ".\serversocket.h"

#define DATA_BUFSIZE 4096         // 接收缓冲区

#define MAX_SOCKET 100            // 最大可以连入的SOCKET数量，这里可以自行更改
                                  // 重叠I/O模型在目前Intel Core级别的主机上可以处理上万个工作量不大的SOCKET连接

#define NULLSOCKET -1

/////////////////////////////////////////////////////////////////////////////
//	如下变量最好应该写入到类的成员变量中去，这里为了降低程序复杂性，就直接定义全局变量了
//  而且需要注意几个线程变量的之间的同步，这里为了代码精简，就省略了

CWinThread* pServerListenThread = NULL;
CWinThread* pWaitForCompletionThread = NULL;

SOCKET sockListen;                               // 监听SOCKET，用以接收客户端连接

SOCKET sockArray[MAX_SOCKET];                    // 与客户端通信的SOCKET，每建立一个新连接，都需要一个SOCKET去通信

WSAOVERLAPPED AcceptOverlapped[MAX_SOCKET];      // 重叠结构数组，每个连入的SOCKET上的每一个重叠操作都得对应一个

WSABUF DataBuf[MAX_SOCKET];                      // 缓冲区，WSARecv的参数
                                                 // 每个SOCKET都对应一个专用的缓冲区

WSAEVENT EventArray[1];                          // 因为WSAWaitForMultipleEvents() API要求在一个或多个事件对象上等待,
												 //	因此不得不创建一个伪事件对象.
                                                 // 但是这个事件数组已经不是和SOCKET相关联的了
                                                 // 这也是完成例程模型和重叠IO模型之间的唯一区别,详见配套文章

int  nSockIndex = 0,                             // SOCKET序号
     nSockTotal = 0,                             // SOCKET总数
	 nCurSockIndex = 0;                          // 当前完成重叠操作的SOCKET（这是一个需要注意线程同步的变量）

BOOL bNewSocket = FALSE;                         // 是否是第一次投递SOCKET上的WSARecv操作（这是一个需要注意线程同步的变量）

HWND CServerSocket::m_hNotifyWnd = NULL;         // 主对话框句柄，用以发送消息



int GetCurrentSocketIndex(LPWSAOVERLAPPED Overlapped);

void ReleaseSocket(const int);


///////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	系统自动调用的回调函数
//
//				在我们投递的WSARecv操作完成的时候，系统会自动调用这个函数	
//
////////////////////////////////////////////////////////////////////////////////////
void CALLBACK _CompletionRoutine(DWORD Error,
								DWORD BytesTransfered,
								LPWSAOVERLAPPED Overlapped,
								DWORD inFlags)
{
	TRACE("回调CompletionRoutine......");

	nCurSockIndex = GetCurrentSocketIndex(Overlapped);             // 根据传入的重叠结构，
	                                                               // 来寻找究竟是哪个SOCKET上触发了事件

	//////////////////////////////////////////////////////////////////////
	//	错误处理：如果返回如下这些值，可能是对方关闭套接字，或者发生一个严重错误
	if(Error != 0 || BytesTransfered == 0)                         
	{
		ReleaseSocket(nCurSockIndex);

		return;
	}

	// 显示一下网络数据的内容(供测试用)
	TRACE("数据：");
	TRACE(DataBuf[nCurSockIndex].buf);

	//////////////////////////////////////////////////////////////////////////////////
	//	程序执行到这里，说明我们先前投递的WSARecv操作已经胜利完成了！！！^_^
	//	DataBuf结构里面就有客户端传来的数据了！！^_^

	CServerSocket::ShowMessage(nCurSockIndex, CString(DataBuf[nCurSockIndex].buf));
		
	return;
}

///////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	监听端口，接收连入的连接
//
////////////////////////////////////////////////////////////////////////////////////
UINT _ServerListenThread(LPVOID lParam)
{
	TRACE("服务器端监听中.....\n");

	CServerSocket* pServer = (CServerSocket*)lParam;

	SOCKADDR_IN ClientAddr;                                  // 定义一个客户端得地址结构作为参数
	int addr_length=sizeof(ClientAddr);

	while(TRUE)
	{
		SOCKET sockTemp = accept(sockListen, (SOCKADDR*)&ClientAddr, &addr_length); 
		if(sockTemp  == INVALID_SOCKET)                       // 检测SOCKET是否有效
		{
			AfxMessageBox("Accept Connection failed！");
			continue;
		}

		nSockIndex = pServer->GetEmptySocket();               // 获得一个空闲的SOCKET索引号

		sockArray[nSockIndex] = sockTemp;                     // 将这个SOCKET句柄存入到数组中

		// 这里可以取得客户端的IP和端口，但是我们只取其中的SOCKET编号(没用的代码我就先屏蔽了)
		//LPCTSTR lpIP =  inet_ntoa(ClientAddr.sin_addr);     // IP
		//UINT nPort = ntohs(ClientAddr.sin_port);            // PORT

		CServerSocket::ShowMessage(nSockIndex,"客户端建立连接！");

		nSockTotal++;                                         // SOCKET总数加一

		bNewSocket = TRUE;                                    // 标志投递一个新的WSARecv请求

		if(nSockTotal >= 1)                                   // SOCKET数量不为空时激活处理线程
			pWaitForCompletionThread->ResumeThread();
	}

	return 0;
}

////////////////////////////////////////////////////////////////////////////////////////////
// 
//	Purposes:	用于投递第一个WSARecv请求，并等待系统完成的通知，然后继续投递后续的请求
//
////////////////////////////////////////////////////////////////////////////////////////////
UINT _WaitForCompletionThread(LPVOID lParam)
{
	TRACE("开始等待线程....\n");

	EventArray[0] = WSACreateEvent();                        // 建立一个事件

	DWORD dwRecvBytes = 0,                                   // WSARecv的参数
		  Flags = 0;

	while(TRUE)
	{
		if(bNewSocket)                                       // 如果标志为True，则表示投递第一个WSARecv请求！
		{
			bNewSocket = FALSE;

		//************************************************************************
		//
		//				现在开始投递第一个WSARecv请求！(仅针对NewSocket) 
		//
		//************************************************************************

			Flags = 0;

			ZeroMemory(&AcceptOverlapped[nSockIndex],sizeof(WSAOVERLAPPED));

			char buffer[DATA_BUFSIZE];
			ZeroMemory(buffer,DATA_BUFSIZE);

			DataBuf[nSockIndex].len = DATA_BUFSIZE;
			DataBuf[nSockIndex].buf = buffer;

			// 将WSAOVERLAPPED结构指定为一个参数,在套接字上投递一个异步WSARecv()请求
			// 并提供下面的作为完成例程的_CompletionRoutine回调函数
			if(WSARecv(sockArray[nSockIndex], &DataBuf[nSockIndex], 1, &dwRecvBytes,
				&Flags, &AcceptOverlapped[nSockIndex], _CompletionRoutine) == SOCKET_ERROR)
			{
				if(WSAGetLastError() != WSA_IO_PENDING)
				{
					ReleaseSocket(nSockIndex);

					continue;
				}
			}
		}

		////////////////////////////////////////////////////////////////////////////////
		// 等待重叠请求完成，自动回调完成例程函数(这里也可以用SleepEx代替)
		DWORD dwIndex = WSAWaitForMultipleEvents(1,EventArray,FALSE,WSA_INFINITE,TRUE);

		///////////////////////////////////////////////////////////////////////////////////
		// 返回WAIT_IO_COMPLETION表示一个重叠请求完成例程代码的结束。继续为更多的完成例程服务
		if(dwIndex == WAIT_IO_COMPLETION)
		{
			
			TRACE("重叠操作完成...\n");

			//************************************************************************
			//
			//				现在开始投递后续的WSARecv请求！ 
			//
			//**********************************************************************

			// 前一个完成例程结束以后，开始在此套接字上投递下一个WSARecv，代码和前面的一模一样^_^

			if(nCurSockIndex != NULLSOCKET)                          // 这个nCurSockIndex来自于前面完成例程得到的那个
			{
				Flags = 0;

				ZeroMemory(&AcceptOverlapped[nCurSockIndex],sizeof(WSAOVERLAPPED));

				char buffer[DATA_BUFSIZE];
				ZeroMemory(buffer,DATA_BUFSIZE);

				DataBuf[nCurSockIndex].len = DATA_BUFSIZE;
				DataBuf[nCurSockIndex].buf = buffer;

				/////////////////////////////////////////////////////////////////////////////////////
				// 将WSAOVERLAPPED结构制定为一个参数,在套接字上投递一个异步WSARecv()请求
				// 并提供下面作为完成例程的CompletionRoutine回调函数
				if(WSARecv(sockArray[nCurSockIndex], &DataBuf[nCurSockIndex], 1, &dwRecvBytes,
					&Flags, &AcceptOverlapped[nCurSockIndex], _CompletionRoutine) == SOCKET_ERROR)
				{
					if(WSAGetLastError() != WSA_IO_PENDING)          // 出现错误，关闭SOCKET
					{
						ReleaseSocket(nCurSockIndex);
						continue;
					}
				}
				// TODO:这里最好添加一个步骤，去掉数组中无效的SOCKET
				// 因为非正常关闭的客户端，比如拔掉网线等，这里是不会接到通知的
			}

			continue;
		}
		else  
		{
			if(dwIndex == WAIT_TIMEOUT)                              // 继续等待
				continue;
			else                                                     // 如果出现其他错误就坏事了 
			{ 
				AfxMessageBox("_WaitForCompletionThread 发生异常，线程将退出！");
				break;
			}
		}
	}

	return 0;
}

////////////////////////////////////////////////////////////////////////////////////
// 
//	Purposes:	构造函数，初始化SOCKET数组
//
////////////////////////////////////////////////////////////////////////////////////
CServerSocket::CServerSocket(void)
{
	for(int i=0;i<MAX_SOCKET;i++)
	{
		sockArray[i] = INVALID_SOCKET;            
	}
}

////////////////////////////////////////////////////////////////////////////////////
// 
//	Purposes:	析构，释放资源
//
////////////////////////////////////////////////////////////////////////////////////
CServerSocket::~CServerSocket(void)
{
	TRACE("CServerSocket 析构！\n");

	StopListening();

	WSACleanup();
}


//////////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	开始监听过程
//
//				初始化监听SOCKET，绑定地址，启动监听线程，都是老套路，不用多说:)
//
///////////////////////////////////////////////////////////////////////////////////////
int CServerSocket::StartListening(const UINT& port)
{
	m_nPort = port;

	WSADATA wsaData;

	int nRet;

	nRet=WSAStartup(MAKEWORD(2,2),&wsaData);                      
	if(nRet!=0)
	{
		AfxMessageBox("Load winsock2 failed");
		WSACleanup();
		return -1;
	}

	sockListen = socket(AF_INET,SOCK_STREAM,IPPROTO_TCP);             //创建服务套接字(TCP)

	SOCKADDR_IN ServerAddr;                                           //分配端口及协议族并绑定

	ServerAddr.sin_family=AF_INET;                                
	ServerAddr.sin_addr.S_un.S_addr  =htonl(INADDR_ANY);          
	ServerAddr.sin_port=htons(port);

	nRet=bind(sockListen,(LPSOCKADDR)&ServerAddr,sizeof(ServerAddr)); // 绑定套接字

	if(nRet==SOCKET_ERROR)
	{
		AfxMessageBox("Bind Socket Fail!");
		closesocket(sockListen);
		return -1;
	}

	nRet=listen(sockListen,1);                                        //开始监听，并设置监听客户端数量
	if(nRet==SOCKET_ERROR)     
	{
		AfxMessageBox("Listening Fail!");
		return -1;
	}


	pServerListenThread =  AfxBeginThread(                            // 启动监听线程
		_ServerListenThread, 
		this
		);  

	pWaitForCompletionThread = AfxBeginThread(                        // 启动完成例程处理线程
		_WaitForCompletionThread,
		this,
		THREAD_PRIORITY_NORMAL,
		0,
		CREATE_SUSPENDED,NULL
		);

	return 0;
}

/////////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	停止监听，关闭线程，释放资源
//
///////////////////////////////////////////////////////////////////////////////////////
int CServerSocket::StopListening()
{
	if(pServerListenThread != NULL)
	{
		DWORD   dwStatus;
		::GetExitCodeThread(pServerListenThread->m_hThread, &dwStatus);

		if (dwStatus == STILL_ACTIVE)
		{
			delete pServerListenThread;
			pServerListenThread = NULL;
		}
	}

	if(pWaitForCompletionThread != NULL)
	{
		DWORD   dwStatus;
		::GetExitCodeThread(pWaitForCompletionThread->m_hThread, &dwStatus);

		if (dwStatus == STILL_ACTIVE)
		{
			delete pWaitForCompletionThread;
			pWaitForCompletionThread = NULL;
		}
	}


	return 0;
}

/////////////////////////////////////////////////////////////////////////////////////////////
//	
//	Purposes:	枚举SOCKET数组，获得一个空闲的SOCKET的索引号
//
//	Hints:		可以利用自己定义的数据结构来获得更好的算法，但是这里考虑到代码复杂性，则略过
//
////////////////////////////////////////////////////////////////////////////////////////////
int CServerSocket::GetEmptySocket()
{
	for(int i=0;i<MAX_SOCKET;i++)
	{
		if(sockArray[i]==INVALID_SOCKET)
		{
			return i;
		}
	}

	return -1;
}

///////////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	根据回调函数中的Overlapped参数，判断是哪个SOCKET上发生了事件
//
//	Hints:		其实有更好的改进算法，但是为了降低代码的复杂性，就留给读者去考虑了
//
///////////////////////////////////////////////////////////////////////////////////////
int GetCurrentSocketIndex(LPWSAOVERLAPPED Overlapped)
{
	for(int i=0;i<nSockTotal;i++)
	{
		if(AcceptOverlapped[i].Internal == Overlapped->Internal)     // 因为每个Overlapped结构的Internal值都是不一样的
			return i;
	}

	return -1;
}

////////////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	给主窗体发消息显示文字信息
//
//	Parameters:	index:	标志哪个SOCKET上发生了事件
//				str		要显示的信息
//
/////////////////////////////////////////////////////////////////////////////////////////
void CServerSocket::ShowMessage(const int index, const CString& str)
{
	CString strSock;
	
	strSock.Format("SOCKET编号:%d", sockArray[index] );

	::SendMessage(
		CServerSocket::m_hNotifyWnd,
		WM_MSG_NEW_SOCKET,
		(LPARAM)(LPCTSTR)strSock,
		(LPARAM)(LPCTSTR)str
		);
}

////////////////////////////////////////////////////////////////////////////////////////
//
//	Purposes:	关闭指定SOCKET，释放资源
//
/////////////////////////////////////////////////////////////////////////////////////////
void ReleaseSocket(const int index)
{
	CServerSocket::ShowMessage(index, CString("客户端断开连接！"));

	closesocket(sockArray[index]);                             // 关闭SOCKET 
	sockArray[index] = INVALID_SOCKET;

	nSockTotal--;                                             // SOCKET总数减一                                         

	nCurSockIndex = NULLSOCKET;

	if(nSockTotal <= 0)                                        // 如果SOCKET总数为0，则处理线程休眠
		pWaitForCompletionThread->SuspendThread();
}
```

## 6. 完成端口(CompletionPort)

资料：https://blog.csdn.net/PiggyXP/article/details/6922277?spm=1001.2014.3001.5502

### 6.1 完成端口的优点

1. 我想只要是写过或者想要写C/S模式网络服务器端的朋友，都应该或多或少的听过完成端口的大名吧，完成端口会充分利用Windows内核来进行I/O的调度，是用于C/S通信模式中性能最好的网络通信模型，没有之一；甚至连和它性能接近的通信模型都没有。
2. 完成端口和其他网络通信方式最大的区别在哪里呢？

(1) 首先，如果使用“同步”的方式来通信的话，这里说的同步的方式就是说所有的操作都在一个线程内顺序执行完成，这么做缺点是很明显的：因为同步的通信操作会阻塞住来自同一个线程的任何其他操作，只有这个操作完成了之后，后续的操作才可以完成；一个最明显的例子就是咱们在MFC的界面代码中，直接使用阻塞Socket调用的代码，整个界面都会因此而阻塞住没有响应！所以我们不得不为每一个通信的Socket都要建立一个线程，多麻烦？这不坑爹呢么？所以要写高性能的服务器程序，要求通信一定要是异步的。

(2) 各位读者肯定知道，可以使用使用“同步通信(阻塞通信)+多线程”的方式来改善(1)的情况，那么好，想一下，我们好不容易实现了让服务器端在每一个客户端连入之后，都要启动一个新的Thread和客户端进行通信，有多少个客户端，就需要启动多少个线程，对吧；但是由于这些线程都是处于运行状态，所以系统不得不在所有可运行的线程之间进行上下文的切换，我们自己是没啥感觉，但是CPU却痛苦不堪了，因为线程切换是相当浪费CPU时间的，如果客户端的连入线程过多，这就会弄得CPU都忙着去切换线程了，根本没有多少时间去执行线程体了，所以效率是非常低下的，承认坑爹了不？

(3) 而微软提出完成端口模型的初衷，就是为了解决这种"one-thread-per-client"的缺点的，它充分利用内核对象的调度，只使用少量的几个线程来处理和客户端的所有通信，消除了无谓的线程上下文切换，最大限度的提高了网络通信的性能，这种神奇的效果具体是如何实现的请看下文。

3. 完成端口被广泛的应用于各个高性能服务器程序上，例如著名的Apache….如果你想要编写的服务器端需要同时处理的并发客户端连接数量有数百上千个的话，那不用纠结了，就是它了。

### 6.2 完成端口的相关概念

#### 6.2.1 异步通信机制及其几种实现方式的比较

我们从前面的文字中了解到，高性能服务器程序使用异步通信机制是必须的。

而对于异步的概念，为了方便后面文字的理解，这里还是再次简单的描述一下：

异步通信就是在咱们与外部的I/O设备进行打交道的时候，我们都知道外部设备的I/O和CPU比起来简直是龟速，比如硬盘读写、网络通信等等，我们没有必要在咱们自己的线程里面等待着I/O操作完成再执行后续的代码，而是将这个请求交给设备的驱动程序自己去处理，我们的线程可以继续做其他更重要的事情，大体的流程如下图所示:

![](.\pic\异步通信.gif)

我可以从图中看到一个很明显的并行操作的过程，而“同步”的通信方式是在进行网络操作的时候，主线程就挂起了，主线程要等待网络操作完成之后，才能继续执行后续的代码，就是说要么执行主线程，要么执行网络操作，是没法这样并行的；

“异步”方式无疑比 “阻塞模式+多线程”的方式效率要高的多，这也是前者为什么叫“异步”，后者为什么叫“同步”的原因了，因为不需要等待网络操作完成再执行别的操作。

而在Windows中实现异步的机制同样有好几种，而这其中的区别，关键就在于图1中的最后一步“通知应用程序处理网络数据”上了，因为实现操作系统调用设备驱动程序去接收数据的操作都是一样的，关键就是在于如何去通知应用程序来拿数据。

(1) 设备内核对象，使用设备内核对象来协调数据的发送请求和接收数据协调，也就是说通过设置设备内核对象的状态，在设备接收数据完成后，马上触发这个内核对象，然后让接收数据的线程收到通知，但是这种方式太原始了，接收数据的线程为了能够知道内核对象是否被触发了，还是得不停的挂起等待，这简直是根本就没有用嘛，太低级了，有木有？所以在这里就略过不提了，各位读者要是没明白是怎么回事也不用深究了，总之没有什么用。

(2) 事件内核对象，利用事件内核对象来实现I/O操作完成的通知，其实这种方式其实就是我以前写文章的时候提到的《基于事件通知的重叠I/O模型》，这种机制就先进得多，可以同时等待多个I/O操作的完成，实现真正的异步，但是缺点也是很明显的，既然用WaitForMultipleObjects()来等待Event的话，就会受到64个Event等待上限的限制，但是这可不是说我们只能处理来自于64个客户端的Socket，而是这是属于在一个设备内核对象上等待的64个事件内核对象，也就是说，我们在一个线程内，可以同时监控64个重叠I/O操作的完成状态，当然我们同样可以使用多个线程的方式来满足无限多个重叠I/O的需求，比如如果想要支持3万个连接，就得需要500多个线程…用起来太麻烦让人感觉不爽；

(3) 使用APC( Asynchronous Procedure Call，异步过程调用)来完成，这个也就是我以前在文章里提到的《基于完成例程的重叠I/O模型》，这种方式的好处就是在于摆脱了基于事件通知方式的64个事件上限的限制，但是缺点也是有的，就是发出请求的线程必须得要自己去处理接收请求，哪怕是这个线程发出了很多发送或者接收数据的请求，但是其他的线程都闲着…，这个线程也还是得自己来处理自己发出去的这些请求，没有人来帮忙…这就有一个负载均衡问题，显然性能没有达到最优化。

(4) 完成端口，不用说大家也知道了，最后的压轴戏就是使用完成端口，对比上面几种机制，完成端口的做法是这样的：事先开好几个线程，你有几个CPU我就开几个，首先是避免了线程的上下文切换，因为线程想要执行的时候，总有CPU资源可用，然后让这几个线程等着，等到有用户请求来到的时候，就把这些请求都加入到一个公共消息队列中去，然后这几个开好的线程就排队逐一去从消息队列中取出消息并加以处理，这种方式就很优雅的实现了异步通信和负载均衡的问题，因为它提供了一种机制来使用几个线程“公平的”处理来自于多个客户端的输入/输出，并且线程如果没事干的时候也会被系统挂起，不会占用CPU周期，挺完美的一个解决方案，不是吗？哦，对了，这个关键的作为交换的消息队列，就是完成端口。

比较完毕之后，熟悉网络编程的朋友可能会问到，为什么没有提到WSAAsyncSelect或者是WSAEventSelect这两个异步模型呢，对于这两个模型，我不知道其内部是如何实现的，但是这其中一定没有用到Overlapped机制，就不能算作是真正的异步，可能是其内部自己在维护一个消息队列吧，总之这两个模式虽然实现了异步的接收，但是却不能进行异步的发送，这就很明显说明问题了，我想其内部的实现一定和完成端口是迥异的，并且，完成端口非常厚道，因为它是先把用户数据接收回来之后再通知用户直接来取就好了，而WSAAsyncSelect和WSAEventSelect之流只是会接收到数据到达的通知，而只能由应用程序自己再另外去recv数据，性能上的差距就更明显了。

最后，我的建议是，想要使用 基于事件通知的重叠I/O和基于完成例程的重叠I/O的朋友，如果不是特别必要，就不要去使用了，因为这两种方式不仅使用和理解起来也不算简单，而且还有性能上的明显瓶颈，何不就再努力一下使用完成端口呢？

####  6.2.2 重叠结构(OVERLAPPED)

我们从上一小节中得知，要实现异步通信，必须要用到一个很风骚的I/O数据结构，叫重叠结构“Overlapped”，Windows里所有的异步通信都是基于它的，完成端口也不例外。

至于为什么叫Overlapped？Jeffrey Richter的解释是因为“执行I/O请求的时间与线程执行其他任务的时间是重叠(overlapped)的”，从这个名字我们也可能看得出来重叠结构发明的初衷了，对于重叠结构的内部细节我这里就不过多的解释了，就把它当成和其他内核对象一样，不需要深究其实现机制，只要会使用就可以了，想要了解更多重叠结构内部的朋友，请去翻阅Jeffrey Richter的《Windows via C/C++》 5th 的292页。

这里我想要解释的是，这个重叠结构是异步通信机制实现的一个核心数据结构，因为你看到后面的代码你会发现，几乎所有的网络操作例如发送/接收之类的，都会用WSASend()和WSARecv()代替，参数里面都会附带一个重叠结构，这是为什么呢？因为重叠结构我们就可以理解成为是一个网络操作的ID号，也就是说我们要利用重叠I/O提供的异步机制的话，每一个网络操作都要有一个唯一的ID号，因为进了系统内核，里面黑灯瞎火的，也不了解上面出了什么状况，一看到有重叠I/O的调用进来了，就会使用其异步机制，并且操作系统就只能靠这个重叠结构带有的ID号来区分是哪一个网络操作了，然后内核里面处理完毕之后，根据这个ID号，把对应的数据传上去。

#### 6.2.3 完成端口(CompletionPort)

对于完成端口这个概念，我一直不知道为什么它的名字是叫“完成端口”，我个人的感觉应该叫它“完成队列”似乎更合适一些，总之这个“端口”和我们平常所说的用于网络通信的“端口”完全不是一个东西，我们不要混淆了。

首先，它之所以叫“完成”端口，就是说系统会在网络I/O操作“完成”之后才会通知我们，也就是说，我们在接到系统的通知的时候，其实网络操作已经完成了，就是比如说在系统通知我们的时候，并非是有数据从网络上到来，而是来自于网络上的数据已经接收完毕了；或者是客户端的连入请求已经被系统接入完毕了等等，我们只需要处理后面的事情就好了。

各位朋友可能会很开心，什么？已经处理完毕了才通知我们，那岂不是很爽？其实也没什么爽的，那是因为我们在之前给系统分派工作的时候，都嘱咐好了，我们会通过代码告诉系统“你给我做这个做那个，等待做完了再通知我”，只是这些工作是做在之前还是之后的区别而已。

其次，我们需要知道，所谓的完成端口，其实和HANDLE一样，也是一个内核对象，虽然Jeff Richter吓唬我们说：“完成端口可能是最为复杂的内核对象了”，但是我们也不用去管他，因为它具体的内部如何实现的和我们无关，只要我们能够学会用它相关的API把这个完成端口的框架搭建起来就可以了。我们暂时只用把它大体理解为一个容纳网络通信操作的队列就好了，它会把网络操作完成的通知，都放在这个队列里面，咱们只用从这个队列里面取就行了，取走一个就少一个…。

### 6.3 使用完成端口的基本流程

大体上来讲，使用完成端口只用遵循如下几个步骤：

(1) 调用 CreateIoCompletionPort() 函数创建一个完成端口，而且在一般情况下，我们需要且只需要建立这一个完成端口，把它的句柄保存好，我们今后会经常用到它……

(2) 根据系统中有多少个处理器，就建立多少个工作者(为了醒目起见，下面直接说Worker)线程，这几个线程是专门用来和客户端进行通信的，目前暂时没什么工作；

(3) 下面就是接收连入的Socket连接了，这里有两种实现方式：一是和别的编程模型一样，还需要启动一个独立的线程，专门用来accept客户端的连接请求；二是用性能更高更好的异步AcceptEx()请求，因为各位对accept用法应该非常熟悉了，而且网上资料也会很多，所以为了更全面起见，本文采用的是性能更好的AcceptEx，至于两者代码编写上的区别，我接下来会详细的讲。

(4) 每当有客户端连入的时候，我们就还是得调用CreateIoCompletionPort()函数，这里却不是新建立完成端口了，而是把新连入的Socket(也就是前面所谓的设备句柄)，与目前的完成端口绑定在一起。

至此，我们其实就已经完成了完成端口的相关部署工作了，嗯，是的，完事了，后面的代码里我们就可以充分享受完成端口带给我们的巨大优势，坐享其成了，是不是很简单呢？

(5) 例如，客户端连入之后，我们可以在这个Socket上提交一个网络请求，例如WSARecv()，然后系统就会帮咱们乖乖的去执行接收数据的操作，我们大可以放心的去干别的事情了；

(6) 而此时，我们预先准备的那几个Worker线程就不能闲着了， 我们在前面建立的几个Worker就要忙活起来了，都需要分别调用GetQueuedCompletionStatus() 函数在扫描完成端口的队列里是否有网络通信的请求存在(例如读取数据，发送数据等)，一旦有的话，就将这个请求从完成端口的队列中取回来，继续执行本线程中后面的处理代码，处理完毕之后，我们再继续投递下一个网络通信的请求就OK了，如此循环。

关于完成端口的使用步骤，用文字来表述就是这么多了，很简单吧？如果你还是不理解，我再配合一个流程图来表示一下：

![](.\pic\采用accept方式的完成端口.gif)

当然，我这里假设你已经对网络编程的基本套路有了解了，所以略去了很多基本的细节，并且为了配合朋友们更好的理解我的代码，在流程图我标出了一些函数的名字，并且画得非常详细。

另外需要注意的是由于对于客户端的连入有两种方式，一种是普通阻塞的accept，另外一种是性能更好的AcceptEx，为了能够方面朋友们从别的网络编程的方式中过渡，我这里画了两种方式的流程图，方便朋友们对比学习，图a是使用accept的方式，当然配套的源代码我默认就不提供了，如果需要的话，我倒是也可以发上来；图b是使用AcceptEx的，并配有配套的源码。

采用accept方式的流程示意图如下：

![](.\pic\采用AcceptEx方式的完成端口.gif)

两个图中最大的相同点是什么？是的，最大的相同点就是主线程无所事事，闲得蛋疼……

为什么呢？因为我们使用了异步的通信机制，这些琐碎重复的事情完全没有必要交给主线程自己来做了，只用在初始化的时候和Worker线程交待好就可以了，用一句话来形容就是，主线程永远也体会不到Worker线程有多忙，而Worker线程也永远体会不到主线程在初始化建立起这个通信框架的时候操了多少的心……

图a中是由 _AcceptThread()负责接入连接，并把连入的Socket和完成端口绑定，另外的多个_WorkerThread()就负责监控完成端口上的情况，一旦有情况了，就取出来处理，如果CPU有多核的话，就可以多个线程轮着来处理完成端口上的信息，很明显效率就提高了。

图b中最明显的区别，也就是AcceptEx和传统的accept之间最大的区别，就是取消了阻塞方式的accept调用，也就是说，AcceptEx也是通过完成端口来异步完成的，所以就取消了专门用于accept连接的线程，用了完成端口来进行异步的AcceptEx调用；然后在检索完成端口队列的Worker函数中，根据用户投递的完成操作的类型，再来找出其中的投递的Accept请求，加以对应的处理。

这样做的好处在哪里？为什么还要异步的投递AcceptEx连接的操作呢？

首先，我可以很明确的告诉各位，如果短时间内客户端的并发连接请求不是特别多的话，用accept和AcceptEx在性能上来讲是没什么区别的。

按照我们目前主流的PC来讲，如果客户端只进行连接请求，而什么都不做的话，我们的Server只能接收大约3万-4万个左右的并发连接，然后客户端其余的连入请求就只能收到WSAENOBUFS (10055)了，因为系统来不及为新连入的客户端准备资源了。

需要准备什么资源？当然是准备Socket了……虽然我们创建Socket只用一行SOCKET s= socket(…) 这么一行的代码就OK了，但是系统内部建立一个Socket是相当耗费资源的，因为Winsock2是分层的机构体系，创建一个Socket需要到多个Provider之间进行处理，最终形成一个可用的套接字。总之，系统创建一个Socket的开销是相当高的，所以用accept的话，系统可能来不及为更多的并发客户端现场准备Socket了。

而AcceptEx比Accept又强大在哪里呢？是有三点：

 (1) 这个好处是最关键的，是因为AcceptEx是在客户端连入之前，就把客户端的Socket建立好了，也就是说，AcceptEx是先建立的Socket，然后才发出的AcceptEx调用，也就是说，在进行客户端的通信之前，无论是否有客户端连入，Socket都是提前建立好了；而不需要像accept是在客户端连入了之后，再现场去花费时间建立Socket。如果各位不清楚是如何实现的，请看后面的实现部分。

 (2) 相比accept只能阻塞方式建立一个连入的入口，对于大量的并发客户端来讲，入口实在是有点挤；而AcceptEx可以同时在完成端口上投递多个请求，这样有客户端连入的时候，就非常优雅而且从容不迫的边喝茶边处理连入请求了。

 (3) AcceptEx还有一个非常体贴的优点，就是在投递AcceptEx的时候，我们还可以顺便在AcceptEx的同时，收取客户端发来的第一组数据，这个是同时进行的，也就是说，在我们收到AcceptEx完成的通知的时候，我们就已经把这第一组数据接完毕了；但是这也意味着，如果客户端只是连入但是不发送数据的话，我们就不会收到这个AcceptEx完成的通知……这个我们在后面的实现部分，也可以详细看到。

 最后，要有一个心里准备，相比accept，异步的AcceptEx使用起来要麻烦得多……

### 6.4 完成端口的实现详解

#### 6.4.1 创建一个完成端口

我们正常情况下，我们需要且只需要建立这一个完成端口，代码很简单:

```c++
HANDLE m_hIOCompletionPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0 );
```

呵呵，看到CreateIoCompletionPort()的参数不要奇怪，参数就是一个INVALID，一个NULL，两个0…，说白了就是一个-1，三个0……简直就和什么都没传一样，但是Windows系统内部却是好一顿忙活，把完成端口相关的资源和数据结构都已经定义好了(在后面的原理部分我们会看到，完成端口相关的数据结构大部分都是一些用来协调各种网络I/O的队列)，然后系统会给我们返回一个有意义的HANDLE，只要返回值不是NULL，就说明建立完成端口成功了，就这么简单，不是吗？

对于最后一个参数 0，我这里要简单的说两句，这个0可不是一个普通的0，它代表的是NumberOfConcurrentThreads，也就是说，允许应用程序同时执行的线程数量。当然，我们这里为了避免上下文切换，最理想的状态就是每个处理器上只运行一个线程了，所以我们设置为0，就是说有多少个处理器，就允许同时多少个线程运行。

因为比如一台机器只有两个CPU（或者两个核心），如果让系统同时运行的线程多于本机的CPU数量的话，那其实是没有什么意义的事情，因为这样CPU就不得不在多个线程之间执行上下文切换，这会浪费宝贵的CPU周期，反而降低的效率，我们要牢记这个原则。

#### 6.4.2 建立Worker线程

我们前面已经提到，这个Worker线程很重要，是用来具体处理网络请求、具体和客户端通信的线程，而且对于线程数量的设置很有意思，要等于系统中CPU的数量，那么我们就要首先获取系统中CPU的数量，代码如下：


```c++
SYSTEM_INFO si;
GetSystemInfo(&si);
int m_nProcessors = si.dwNumberOfProcessors;
```

我们最好是建立**CPU核心数量*2**那么多的线程，这样更可以充分利用CPU资源，因为完成端口的调度是非常智能的，比如我们的Worker线程有的时候可能会有Sleep()或者WaitForSingleObject()之类的情况，这样同一个CPU核心上的另一个线程就可以代替这个Sleep的线程执行了；因为完成端口的目标是要使得CPU满负荷的工作。

这里也有人说是建立 **CPU核心数量 * 2 +2**个线程，我想这个应该没有什么太大的区别，我就是按照我自己的习惯来了。

Worker线程和普通线程是一样的，代码大致上如下：

```c++
// 根据CPU数量，建立*2的线程
m_nThreads = 2 * m_nProcessors;
HANDLE* m_phWorkerThreads = new HANDLE[m_nThreads];

for (int i = 0; i < m_nThreads; i++)
{
    // 其中，_WorkerThread是Worker线程的线程函数
    m_phWorkerThreads[i] = ::CreateThread(0, 0, _WorkerThread, …);
}
```

#### 6.4.3 创建监听Socket绑定到完成端口上

最重要的完成端口建立完毕了，我们就可以利用这个完成端口来进行网络通信了。

   首先，我们需要初始化Socket，这里和通常情况下使用Socket初始化的步骤都是一样的，大约就是如下的这么几个过程(详情参照代码中的LoadSocketLib()和InitializeListenSocket()，这里只是挑出关键部分)：

```c++
// 初始化Socket库
WSADATA wsaData;
WSAStartup(MAKEWORD(2,2), &wsaData);
//初始化Socket
struct sockaddr_in ServerAddress;
// 这里需要特别注意，如果要使用重叠I/O的话，这里必须要使用WSASocket来初始化Socket
// 注意里面有个WSA_FLAG_OVERLAPPED参数
SOCKET m_sockListen = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
// 填充地址结构信息
ZeroMemory((char *)&ServerAddress, sizeof(ServerAddress));
ServerAddress.sin_family = AF_INET;
// 这里可以选择绑定任何一个可用的地址，或者是自己指定的一个IP地址	
//ServerAddress.sin_addr.s_addr = htonl(INADDR_ANY);                      
ServerAddress.sin_addr.s_addr = inet_addr(“你的IP”);         
ServerAddress.sin_port = htons(11111);                          
// 绑定端口
if (SOCKET_ERROR == bind(m_sockListen, (struct sockaddr *) &ServerAddress, sizeof(ServerAddress))) 
    // 开始监听
    listen(m_sockListen,SOMAXCONN));
```

需要注意的地方有两点：

(1) 想要使用重叠I/O的话，初始化Socket的时候一定要使用WSASocket并带上WSA_FLAG_OVERLAPPED参数才可以(只有在服务器端需要这么做，在客户端是不需要的)；

(2) 注意到listen函数后面用的那个常量SOMAXCONN了吗？这个是在微软在WinSock2.h中定义的，并且还附赠了一条注释，Maximum queue length specifiable by listen.，所以说，不用白不用咯。

接下来有一个非常重要的动作：既然我们要使用完成端口来帮我们进行监听工作，那么我们一定要把这个监听Socket和完成端口绑定才可以的吧，如何绑定呢？同样很简单，用 CreateIoCompletionPort()函数。

等等！大家没觉得这个函数很眼熟么？是的，这个和前面那个创建完成端口用的居然是同一个API！但是这里这个API可不是用来建立完成端口的，而是用于将Socket和以前创建的那个完成端口绑定的，大家可要看准了，不要被迷惑了，因为他们的参数是明显不一样的，前面那个的参数是一个-1，三个0，太好记了…

说实话，我感觉微软应该把这两个函数分开，弄个 CreateNewCompletionPort() 多好呢？

这里在详细讲解一下CreateIoCompletionPort()的几个参数：

```c
HANDLE WINAPI CreateIoCompletionPort(
    __in      HANDLE  FileHandle,             // 这里当然是连入的这个套接字句柄了
    __in_opt  HANDLE  ExistingCompletionPort, // 这个就是前面创建的那个完成端口
    // 这个参数就是类似于线程参数一样，在绑定的时候把自己定义的结构体指针传递，这样到了Worker线程中，也可以使用这个结构体的数据了，相当于参数的传递
    __in      ULONG_PTR CompletionKey,
    __in      DWORD NumberOfConcurrentThreads // 这里同样置0
);
```

#### 6.4.4 在这个监听Socket上投递AcceptEx请求

这个AcceptEx比较特别，而且这个是微软专门在Windows操作系统里面提供的扩展函数，也就是说这个不是Winsock2标准里面提供的，是微软为了方便咱们使用重叠I/O机制，额外提供的一些函数，所以在使用之前也还是需要进行些准备工作。

微软的实现是通过mswsock.dll中提供的，所以我们可以通过静态链接mswsock.lib来使用AcceptEx。但是这是一个不推荐的方式，**我们应该用WSAIoctl 配合SIO_GET_EXTENSION_FUNCTION_POINTER参数来获取函数的指针，然后再调用AcceptEx**。

这是为什么呢？因为我们在未取得函数指针的情况下就调用AcceptEx的开销是很大的，因为AcceptEx 实际上是存在于Winsock2结构体系之外的(因为是微软另外提供的)，所以如果我们直接调用AcceptEx的话，首先我们的代码就只能在微软的平台上用了，没有办法在其他平台上调用到该平台提供的AcceptEx的版本(如果有的话)， 而且更糟糕的是，我们每次调用AcceptEx时，Service Provider都得要通过WSAIoctl()获取一次该函数指针，效率太低了，所以还不如我们自己直接在代码中直接去这么获取一下指针好了。

获取AcceptEx函数指针的代码大致如下：

```c++
LPFN_ACCEPTEX m_lpfnAcceptEx; // AcceptEx函数指针
GUID GuidAcceptEx = WSAID_ACCEPTEX; // GUID，这个是识别AcceptEx函数必须的
DWORD dwBytes = 0;  
 
WSAIoctl(m_pListenContext->m_Socket, 
         SIO_GET_EXTENSION_FUNCTION_POINTER, 
         &GuidAcceptEx, 
         sizeof(GuidAcceptEx), 
         &m_lpfnAcceptEx, 
         sizeof(m_lpfnAcceptEx), 
         &dwBytes, 
         NULL, 
         NULL);
```

具体实现就没什么可说的了，因为都是固定的套路，那个GUID是微软给定义好的，直接拿过来用就行了，WSAIoctl()就是通过这个找到AcceptEx的地址的，另外需要注意的是，通过WSAIoctl获取AcceptEx函数指针时，只需要随便传递给WSAIoctl()一个有效的SOCKET即可，该Socket的类型不会影响获取的AcceptEx函数指针。

然后，我们就可以通过其中的指针m_lpfnAcceptEx调用AcceptEx函数了。AcceptEx函数的定义如下：

```c++
BOOL AcceptEx ( 	
    SOCKET sListenSocket, 
    SOCKET sAcceptSocket, 
    PVOID lpOutputBuffer, 
    DWORD dwReceiveDataLength, 
    DWORD dwLocalAddressLength, 
    DWORD dwRemoteAddressLength, 
    LPDWORD lpdwBytesReceived, 
    LPOVERLAPPED lpOverlapped 
);
```

乍一看起来参数很多，但是实际用起来也很简单：

参数1--sListenSocket, 这个就是那个唯一的用来监听的Socket了，没什么说的；

参数2--sAcceptSocket, 用于接受连接的socket，这个就是那个需要我们事先建好的，等有客户端连接进来直接把这个Socket拿给它用的那个，是AcceptEx高性能的关键所在。

参数3--lpOutputBuffer,接收缓冲区， 这也是AcceptEx比较有特色的地方，既然AcceptEx不是普通的accpet函数，那么这个缓冲区也不是普通的缓冲区，这个缓冲区包含了三个信息：一是客户端发来的第一组数据，二是server的地址，三是client地址，都是精华啊…但是读取起来就会很麻烦，不过后面有一个更好的解决方案。

参数4--dwReceiveDataLength，前面那个参数lpOutputBuffer中用于存放数据的空间大小。如果此参数=0，则Accept时将不会待数据到来，而直接返回，如果此参数不为0，那么一定得等接收到数据了才会返回…… 所以通常当需要Accept接收数据时，就需要将该参数设成为：sizeof(lpOutputBuffer) - 2*(sizeof sockaddr_in +16)，也就是说总长度减去两个地址空间的长度就是了，看起来复杂，其实想明白了也没啥……

参数5--dwLocalAddressLength，存放本地 地址信息的空间大小；

参数6--dwRemoteAddressLength，存放远端 地址信息的空间大小；

参数7--lpdwBytesReceived，out参数，对我们来说没用，不用管；

参数8--lpOverlapped，本次重叠I/O所要用到的重叠结构。

​        这里面的参数倒是没什么，看起来复杂，但是咱们依旧可以一个一个传进去，然后在对应的IO操作完成之后，这些参数Windows内核自然就会帮咱们填满了。

但是非常悲催的是，我们这个是异步操作，我们是在线程启动的地方投递的这个操作， 等我们再次见到这些个变量的时候，就已经是在Worker线程内部了，因为Windows会直接把操作完成的结果传递到Worker线程里，这样咱们在启动的时候投递了那么多的IO请求，这从Worker线程传回来的这些结果，到底是对应着哪个IO请求的呢？

聪明的你肯定想到了，是的，Windows内核也帮我们想到了：用一个标志来绑定每一个IO操作，这样到了Worker线程内部的时候，收到网络操作完成的通知之后，再通过这个标志来找出这组返回的数据到底对应的是哪个Io操作的。

这里的标志就是如下这样的结构体：

```c++
typedef struct _PER_IO_CONTEXT {
    OVERLAPPED   m_Overlapped;          // 每一个重叠I/O网络操作都要有一个              
    SOCKET       m_sockAccept;          // 这个I/O操作所使用的Socket，每个连接的都是一样的
    WSABUF       m_wsaBuf;              // 存储数据的缓冲区，用来给重叠操作传递参数的，关于WSABUF后面还会讲
    char         m_szBuffer[MAX_BUFFER_LEN]; // 对应WSABUF里的缓冲区
    OPERATION_TYPE  m_OpType;               // 标志这个重叠I/O操作是做什么的，例如Accept/Recv等
} PER_IO_CONTEXT, *PPER_IO_CONTEXT;
```

这个结构体的成员当然是我们随便定义的，里面的成员你可以随意修改(除了OVERLAPPED那个之外……)。

但是AcceptEx不是普通的accept，buffer不是普通的buffer，那么这个结构体当然也不能是普通的结构体了……

在完成端口的世界里，这个结构体有个专属的名字“单IO数据”，是什么意思呢？也就是说每一个重叠I/O都要对应的这么一组参数，至于这个结构体怎么定义无所谓，而且这个结构体也不是必须要定义的，但是没它……还真是不行，我们可以把它理解为线程参数，就好比你使用线程的时候，线程参数也不是必须的，但是不传还真是不行……

除此以外，我们也还会想到，既然每一个I/O操作都有对应的PER_IO_CONTEXT结构体，而在每一个Socket上，我们会投递多个I/O请求的，例如我们就可以在监听Socket上投递多个AcceptEx请求，所以同样的，我们也还需要一个“单句柄数据”来管理这个句柄上所有的I/O请求，这里的“句柄”当然就是指的Socket了，我在代码中是这样定义的：

```c++
typedef struct _PER_SOCKET_CONTEXT {
    SOCKET                   m_Socket;     // 每一个客户端连接的Socket
    SOCKADDR_IN              m_ClientAddr; // 这个客户端的地址
    // 数组，所有客户端IO操作的参数，也就是说对于每一个客户端Socket是可以在上面同时投递多个IO请求的
    CArray<_PER_IO_CONTEXT*> m_arrayIoContext;
} PER_SOCKET_CONTEXT, *PPER_SOCKET_CONTEXT;
```

这也是比较好理解的，也就是说我们需要在一个Socket句柄上，管理在这个Socket上投递的每一个IO请求的_PER_IO_CONTEXT。

当然，同样的，对于这些也可以按照自己的想法来随便定义，只要能起到管理每一个IO请求上需要传递的网络参数的目的就好了，关键就是需要跟踪这些参数的状态，在必要的时候释放这些资源，不要造成内存泄漏，因为作为Server总是需要长时间运行的，所以如果有内存泄露的情况那是非常可怕的，一定要杜绝一丝一毫的内存泄漏。至于具体这两个结构体参数是如何在Worker线程里大发神威的，我们后面再看。

完成端口初始化的工作比起其他的模型来讲是要更复杂一些，所以说对于主线程来讲，它总觉得自己付出了很多，总觉得Worker线程是坐享其成，但是Worker自己的苦只有自己明白，Worker线程的工作一点也不比主线程少，相反还要更复杂一些，并且具体的通信工作全部都是Worker线程来完成的，Worker线程反而还觉得主线程是在旁边看热闹，只知道发号施令而已，但是大家终究还是谁也离不开谁，这也就和公司里老板和员工的微妙关系是一样的吧……

#### 6.4.5 Worker线程都做了些什么

_Worker线程的工作都是涉及到具体的通信事务问题，主要完成了如下的几个工作，让我们一步一步的来看。

(1) 使用 GetQueuedCompletionStatus() 监控完成端口

首先这个工作所要做的工作大家也能猜到，无非就是几个Worker线程哥几个一起排好队队来监视完成端口的队列中是否有完成的网络操作就好了，代码大体如下：

```c++
void       *lpContext = NULL;
OVERLAPPED *pOverlapped = NULL;
DWORD      dwBytesTransfered = 0;

BOOL bReturn  =  GetQueuedCompletionStatus(
	                      	 pIOCPModel->m_hIOCompletionPort,
                             &dwBytesTransfered,
			                 (LPDWORD)&lpContext,
			                 &pOverlapped,
			                 INFINITE);
```

各位留意到其中的GetQueuedCompletionStatus()函数了吗？这个就是Worker线程里第一件也是最重要的一件事了，这个函数的作用就是我在前面提到的，会让Worker线程进入不占用CPU的睡眠状态，直到完成端口上出现了需要处理的网络操作或者超出了等待的时间限制为止。一旦完成端口上出现了已完成的I/O请求，那么等待的线程会被立刻唤醒，然后继续执行后续的代码。

至于这个神奇的函数，原型是这样的：

```C++
BOOL WINAPI GetQueuedCompletionStatus(
    __in   HANDLE          CompletionPort,    // 这个就是我们建立的那个唯一的完成端口
    __out  LPDWORD         lpNumberOfBytes,   // 这个是操作完成后返回的字节数
    __out  PULONG_PTR      lpCompletionKey,   // 这个是我们建立完成端口的时候绑定的那个自定义结构体参数
    __out  LPOVERLAPPED    *lpOverlapped,     // 这个是我们在连入Socket的时候一起建立的那个重叠结构
    __in   DWORD           dwMilliseconds     // 等待完成端口的超时时间，如果线程不需要做其他的事情，那就INFINITE就行了
);
```

所以，如果这个函数突然返回了，那就说明有需要处理的网络操作了 --- 当然，在没有出现错误的情况下。

然后switch()一下，根据需要处理的操作类型，那我们来进行相应的处理。

但是如何知道操作是什么类型的呢？这就需要用到从外部传递进来的loContext参数，也就是我们封装的那个参数结构体，这个参数结构体里面会带有我们一开始投递这个操作的时候设置的操作类型，然后我们根据这个操作再来进行对应的处理。

但是还有问题，这个参数究竟是从哪里传进来的呢？传进来的时候内容都有些什么？

首先，我们要知道两个关键点：

(1) 这个参数，是在你绑定Socket到一个完成端口的时候，用的CreateIoCompletionPort()函数，传入的那个CompletionKey参数，要是忘了的话，就翻到文档的“第三步”看看相关的内容；我们在这里传入的是定义的PER_SOCKET_CONTEXT，也就是说“单句柄数据”，因为我们绑定的是一个Socket，这里自然也就需要传入Socket相关的上下文，你是怎么传过去的，这里收到的就会是什么样子，也就是说这个lpCompletionKey就是我们的PER_SOCKET_CONTEXT，直接把里面的数据拿出来用就可以了。

   (2) 另外还有一个很神奇的地方，里面的那个lpOverlapped参数，里面就带有我们的PER_IO_CONTEXT。这个参数是从哪里来的呢？我们去看看前面投递AcceptEx请求的时候，是不是传了一个重叠参数进去？这里就是它了，并且，我们可以使用一个很神奇的宏，把和它存储在一起的其他的变量，全部都读取出来，例如：

```c++
PER_IO_CONTEXT* pIoContext = CONTAINING_RECORD(lpOverlapped, PER_IO_CONTEXT, m_Overlapped);
```

这个宏的含义，就是去传入的lpOverlapped变量里，找到和结构体中PER_IO_CONTEXT中m_Overlapped成员相关的数据。

你仔细想想，其实真的很神奇……

但是要做到这种神奇的效果，应该确保我们在结构体PER_IO_CONTEXT定义的时候，把Overlapped变量，定义为结构体中的第一个成员。

只要能弄清楚这个GetQueuedCompletionStatus()中各种奇怪的参数，那我们就离成功不远了。

 既然我们可以获得PER_IO_CONTEXT结构体，那么我们就自然可以根据其中的m_OpType参数，得知这次收到的这个完成通知，是关于哪个Socket上的哪个I/O操作的，这样就分别进行对应处理就好了。

在示例代码里，在有AcceptEx请求完成的时候，我是执行的_DoAccept()函数，在有WSARecv请求完成的时候，执行的是_DoRecv()函数，下面我就分别讲解一下这两个函数的执行流程。

#### 6.4.6 当收到Accept通知时 _DoAccept()

在用户收到AcceptEx的完成通知时，需要后续代码并不多，但却是逻辑最为混乱，最容易出错的地方，这也是很多用户为什么宁愿用效率低下的accept()也不愿意去用AcceptEx的原因吧。和普通的Socket通讯方式一样，在有客户端连入的时候，我们需要做三件事情：

   (1) 为这个新连入的连接分配一个Socket；

   (2) 在这个Socket上投递第一个异步的发送/接收请求；

   (3) 继续监听。

其实都是一些很简单的事情但是由于“单句柄数据”和“单IO数据”的加入，事情就变得比较乱。因为是这样的，让我们一起缕一缕啊，最好是配合代码一起看，否则太抽象了……

(1) 首先，_Worker线程通过GetQueuedCompletionStatus()里会收到一个lpCompletionKey，这个也就是PER_SOCKET_CONTEXT，里面保存了与这个I/O相关的Socket和Overlapped还有客户端发来的第一组数据等等，对吧？但是这里得注意，这个SOCKET的上下文数据，是关于监听Socket的，而不是新连入的这个客户端Socket的，千万别弄混了……

(2) 所以，AcceptEx不是给咱们新连入的这个Socket早就建好了一个Socket吗？所以这里，我们需要再用这个新Socket重新为新客户端建立一个PER_SOCKET_CONTEXT，以及下面一系列的新PER_IO_CONTEXT，千万不要去动传入的这个Listen Socket上的PER_SOCKET_CONTEXT，也不要用传入的这个Overlapped信息，因为这个是属于AcceptEx I/O操作的，也不是属于你投递的那个Recv I/O操作的……，要不你下次继续监听的时候就悲剧了……

(3) 等到新的Socket准备完毕了，我们就赶紧还是用传入的这个Listen Socket上的PER_SOCKET_CONTEXT和PER_IO_CONTEXT去继续投递下一个AcceptEx，循环起来，留在这里太危险了，早晚得被人给改了……

(4) 而我们新的Socket的上下文数据和I/O操作数据都准备好了之后，我们要做两件事情：一件事情是把这个新的Socket和我们唯一的那个完成端口绑定，这个就不用细说了，和前面绑定监听Socket是一样的；然后就是在这个Socket上投递第一个I/O操作请求，在我的示例代码里投递的是WSARecv()。因为后续的WSARecv，就不是在这里投递的了，这里只负责第一个请求。

但是，至于WSARecv请求如何来投递的，我们放到下一节中去讲，这一节，我们还有一个很重要的事情，我得给大家提一下，就是在客户端连入的时候，我们如何来获取客户端的连入地址信息。

这里我们还需要引入另外一个很高端的函数，GetAcceptExSockAddrs()，它和AcceptEx()一样，都是微软提供的扩展函数，所以同样需要通过下面的方式来导入才可以使用……

```c++
WSAIoctl(
    m_pListenContext->m_Socket, 
    SIO_GET_EXTENSION_FUNCTION_POINTER, 
    &GuidGetAcceptExSockAddrs,
    sizeof(GuidGetAcceptExSockAddrs), 
    &m_lpfnGetAcceptExSockAddrs, 
    sizeof(m_lpfnGetAcceptExSockAddrs),   
    &dwBytes, 
    NULL, 
    NULL);
```

和导出AcceptEx一样一样的，同样是需要用其GUID来获取对应的函数指针 m_lpfnGetAcceptExSockAddrs 。

说了这么多，这个函数究竟是干嘛用的呢？它是名副其实的“AcceptEx之友”，为什么这么说呢？因为我前面提起过AcceptEx有个很神奇的功能，就是附带一个神奇的缓冲区，这个缓冲区厉害了，包括了客户端发来的第一组数据、本地的地址信息、客户端的地址信息，三合一啊，你说神奇不神奇？

这个函数从它字面上的意思也基本可以看得出来，就是用来解码这个缓冲区的，是的，它不提供别的任何功能，就是专门用来解析AcceptEx缓冲区内容的。例如如下代码：

```c++
PER_IO_CONTEXT* pIoContext = 本次通信用的I/O Context
 
SOCKADDR_IN* ClientAddr = NULL;
SOCKADDR_IN* LocalAddr = NULL;  
int remoteLen = sizeof(SOCKADDR_IN), localLen = sizeof(SOCKADDR_IN);  
 
m_lpfnGetAcceptExSockAddrs(pIoContext->m_wsaBuf.buf, pIoContext->m_wsaBuf.len - ((sizeof(SOCKADDR_IN)+16)*2),  sizeof(SOCKADDR_IN)+16, sizeof(SOCKADDR_IN)+16, (LPSOCKADDR*)&LocalAddr, &localLen, (LPSOCKADDR*)&ClientAddr, &remoteLen);
```

 解码完毕之后，于是，我们就可以从如下的结构体指针中获得很多有趣的地址信息了：

> inet_ntoa(ClientAddr->sin_addr) 是客户端IP地址
>
> ntohs(ClientAddr->sin_port) 是客户端连入的端口
>
> inet_ntoa(LocalAddr ->sin_addr) 是本地IP地址
>
> ntohs(LocalAddr ->sin_port) 是本地通讯的端口
>
> pIoContext->m_wsaBuf.buf 是存储客户端发来第一组数据的缓冲区
>

#### 6.4.7 当收到Recv通知时, _DoRecv()

在讲解如何处理Recv请求之前，我们还是先讲一下如何投递WSARecv请求的。

WSARecv大体的代码如下，其实就一行，在代码中我们可以很清楚的看到我们用到了很多新建的PerIoContext的参数，这里再强调一下，注意一定要是自己另外新建的啊，一定不能是Worker线程里传入的那个PerIoContext，因为那个是监听Socket的，别给人弄坏了……：

```c++
int nBytesRecv = WSARecv(pIoContext->m_Socket, pIoContext ->p_wbuf, 1, &dwBytes, 0, pIoContext->p_ol, NULL);
```

这里，我再把WSARev函数的原型再给各位讲一下

```c++
int WSARecv(
    SOCKET s,                      // 当然是投递这个操作的套接字
    LPWSABUF lpBuffers,            // 接收缓冲区，这里需要一个由WSABUF结构构成的数组
    DWORD dwBufferCount,           // 数组中WSABUF结构的数量，设置为1即可
    LPDWORD lpNumberOfBytesRecvd,  // 如果接收操作立即完成，这里会返回函数调用所接收到的字节数
    LPDWORD lpFlags,               // 说来话长了，我们这里设置为0 即可
    LPWSAOVERLAPPED lpOverlapped,  // 这个Socket对应的重叠结构
    NULL                           // 这个参数只有完成例程模式才会用到，完成端口中我们设置为NULL即可
);

       
typedef struct _WSABUF {
    ULONG len; /* the length of the buffer */
    __field_bcount(len) CHAR FAR *buf; /* the pointer to the buffer */
} WSABUF, FAR * LPWSABUF;
```

其实里面的参数，如果你们熟悉或者看过我以前的重叠I/O的文章，应该都比较熟悉，只需要注意其中的两个参数：

- `LPWSABUF lpBuffers`这里是需要我们自己new 一个 WSABUF 的结构体传进去的，在ws2def.h中定义。

这里需要注意的，我们的应用程序接到数据到达的通知的时候，其实数据已经被咱们的主机接收下来了，我们直接通过这个WSABUF指针去系统缓冲区拿数据就好了，而不像那些没用重叠I/O的模型，接收到有数据到达的通知的时候还得自己去另外recv，太低端了……这也是为什么重叠I/O比其他的I/O性能要好的原因之一。

- `LPWSAOVERLAPPED *lpOverlapped`这个参数就是我们所谓的重叠结构了，就是这样定义，然后在有Socket连接进来的时候，生成并初始化一下，然后在投递第一个完成请求的时候，作为参数传递进去就可以。

```c++
OVERLAPPED* m_pol = new OVERLAPPED;
ZeroMemory(m_pol, sizeof(OVERLAPPED));
```

在第一个重叠请求完毕之后，我们的这个OVERLAPPED 结构体里，就会被分配有效的系统参数了，并且我们是需要每一个Socket上的每一个I/O操作类型，都要有一个唯一的Overlapped结构去标识。

这样，投递一个WSARecv就讲完了，至于_DoRecv()需要做些什么呢？其实就是做两件事：

(1) 把WSARecv里这个缓冲区里收到的数据显示出来；

(2) 发出下一个WSARecv()；

#### 6.4.8 如何关闭完成端口

从前面的章节中，我们已经了解到，Worker线程一旦进入了GetQueuedCompletionStatus()的阶段，就会进入睡眠状态，INFINITE的等待完成端口中，如果完成端口上一直都没有已经完成的I/O请求，那么这些线程将无法被唤醒，这也意味着线程没法正常退出。

熟悉或者不熟悉多线程编程的朋友，都应该知道，如果在线程睡眠的时候，简单粗暴的就把线程关闭掉的话，那是会一个很可怕的事情，因为很多线程体内很多资源都来不及释放掉，无论是这些资源最后是否会被操作系统回收，我们作为一个C++程序员来讲，都不应该允许这样的事情出现。

所以我们必须得有一个很优雅的，让线程自己退出的办法。

这时会用到我们这次见到的与完成端口有关的最后一个API，叫 PostQueuedCompletionStatus()，从名字上也能看得出来，这个是和 GetQueuedCompletionStatus() 函数相对的，这个函数的用途就是可以让我们手动的添加一个完成端口I/O操作，这样处于睡眠等待的状态的线程就会有一个被唤醒，如果为我们每一个Worker线程都调用一次PostQueuedCompletionStatus()的话，那么所有的线程也就会因此而被唤醒了。PostQueuedCompletionStatus()函数的原型是这样定义的：

```c++
BOOL WINAPI PostQueuedCompletionStatus(
    __in      HANDLE CompletionPort,
    __in      DWORD dwNumberOfBytesTransferred,
    __in      ULONG_PTR dwCompletionKey,
    __in_opt  LPOVERLAPPED lpOverlapped
);
```

   我们可以看到，这个函数的参数几乎和GetQueuedCompletionStatus()的一模一样，都是需要把我们建立的完成端口传进去，然后后面的三个参数是 传输字节数、结构体参数、重叠结构的指针.

注意，这里也有一个很神奇的事情，正常情况下，GetQueuedCompletionStatus()获取回来的参数本来是应该是系统帮我们填充的，或者是在绑定完成端口时就有的，但是我们这里却可以直接使用PostQueuedCompletionStatus()直接将后面三个参数传递给GetQueuedCompletionStatus()，这样就非常方便了。

   例如，我们为了能够实现通知线程退出的效果，可以自己定义一些约定，比如把这后面三个参数设置一个特殊的值，然后Worker线程接收到完成通知之后，通过判断这3个参数中是否出现了特殊的值，来决定是否是应该退出线程了。

   例如我们在调用的时候，就可以这样：

```c++
for (int i = 0; i < m_nThreads; i++)
{
    PostQueuedCompletionStatus(m_hIOCompletionPort, 0, (DWORD) NULL, NULL);
}
```

为每一个线程都发送一个完成端口数据包，有几个线程就发送几遍，把其中的dwCompletionKey参数设置为NULL，这样每一个Worker线程在接收到这个完成通知的时候，再自己判断一下这个参数是否被设置成了NULL，因为正常情况下，这个参数总是会有一个非NULL的指针传入进来的，如果Worker发现这个参数被设置成了NULL，那么Worker线程就会知道，这是应用程序再向Worker线程发送的退出指令，这样Worker线程在内部就可以自己很“优雅”的退出了……

但是这里有一个很明显的问题，聪明的朋友一定想到了，而且只有想到了这个问题的人，才算是真正看明白了这个方法。

我们只是发送了m_nThreads次，我们如何能确保每一个Worker线程正好就收到一个，然后所有的线程都正好退出呢？是的，我们没有办法保证，所以很有可能一个Worker线程处理完一个完成请求之后，发生了某些事情，结果又再次去循环接收下一个完成请求了，这样就会造成有的Worker线程没有办法接收到我们发出的退出通知。

所以，我们在退出的时候，一定要确保Worker线程只调用一次GetQueuedCompletionStatus()，这就需要我们自己想办法了，各位请参考我在Worker线程中实现的代码，我搭配了一个退出的Event，在退出的时候SetEvent一下，来确保Worker线程每次就只会调用一轮 GetQueuedCompletionStatus() ，这样就应该比较安全了。

另外，在Vista/Win7系统中，我们还有一个更简单的方式，我们可以直接CloseHandle关掉完成端口的句柄，这样所有在GetQueuedCompletionStatus()的线程都会被唤醒，并且返回FALSE，这时调用GetLastError()获取错误码时，会返回ERROR_INVALID_HANDLE，这样每一个Worker线程就可以通过这种方式轻松简单的知道自己该退出了。当然，如果我们不能保证我们的应用程序只在Vista/Win7中，那还是老老实实的PostQueuedCompletionStatus()吧。

最后，在系统释放资源的最后阶段，切记，因为完成端口同样也是一个Handle，所以也得用CloseHandle将这个句柄关闭，当然还要记得用closesocket关闭一系列的socket，还有别的各种指针什么的，这都是作为一个合格的C++程序员的基本功，在这里就不多说了，如果还是有不太清楚的朋友，请参考我的示例代码中的 StopListen() 和DeInitialize() 函数。

### 6.5 完成端口使用中的注意事项

1. Socket的通信缓冲区设置成多大合适？

在x86的体系中，内存页面是以4KB为单位来锁定的，也就是说，就算是你投递WSARecv()的时候只用了1KB大小的缓冲区，系统还是得给你分4KB的内存。为了避免这种浪费，最好是把发送和接收数据的缓冲区直接设置成4KB的倍数。

2. 关于完成端口通知的次序问题

 这个不用想也能知道，调用GetQueuedCompletionStatus() 获取I/O完成端口请求的时候，肯定是用先入先出的方式来进行的。

 但是，咱们大家可能都想不到的是，唤醒那些调用了GetQueuedCompletionStatus()的线程是以后入先出的方式来进行的。

 比如有4个线程在等待，如果出现了一个已经完成的I/O项，那么是最后一个调用GetQueuedCompletionStatus()的线程会被唤醒。平常这个次序倒是不重要，但是在对数据包顺序有要求的时候，比如传送大块数据的时候，是需要注意下这个先后次序的。

 -- 微软之所以这么做，那当然是有道理的，这样如果反复只有一个I/O操作而不是多个操作完成的话，内核就只需要唤醒同一个线程就可以了，而不需要轮着唤醒多个线程，节约了资源，而且可以把其他长时间睡眠的线程换出内存，提到资源利用率。

3. 如果各位想要传输文件…

如果各位需要使用完成端口来传送文件的话，这里有个非常需要注意的地方。因为发送文件的做法，按照正常人的思路来讲，都会是先打开一个文件，然后不断的循环调用ReadFile()读取一块之后，然后再调用WSASend ()去发发送。

但是我们知道，ReadFile()的时候，是需要操作系统通过磁盘的驱动程序，到实际的物理硬盘上去读取文件的，这就会使得操作系统从用户态转换到内核态去调用驱动程序，然后再把读取的结果返回至用户态；同样的道理，WSARecv()也会涉及到从用户态到内核态切换的问题 --- 这样就使得我们不得不频繁的在用户态到内核态之间转换，效率低下……

而一个非常好的解决方案是使用微软提供的扩展函数TransmitFile()来传输文件，因为只需要传递给TransmitFile()一个文件的句柄和需要传输的字节数，程序就会整个切换至内核态，无论是读取数据还是发送文件，都是直接在内核态中执行的，直到文件传输完毕才会返回至用户态给主进程发送通知。这样效率就高多了。

4. 关于重叠结构数据释放的问题

 我们既然使用的是异步通讯的方式，就得要习惯一点，就是我们投递出去的完成请求，不知道什么时候我们才能收到操作完成的通知，而在这段等待通知的时间，我们就得要千万注意得保证我们投递请求的时候所使用的变量在此期间都得是有效的。

 例如我们发送WSARecv请求时候所使用的Overlapped变量，因为在操作完成的时候，这个结构里面会保存很多很重要的数据，对于设备驱动程序来讲，指示保存着我们这个Overlapped变量的指针，而在操作完成之后，驱动程序会将Buffer的指针、已经传输的字节数、错误码等等信息都写入到我们传递给它的那个Overlapped指针中去。如果我们已经不小心把Overlapped释放了，或者是又交给别的操作使用了的话，谁知道驱动程序会把这些东西写到哪里去呢？岂不是很崩溃……

###  6.6 例子

#### 1. 服务器IOCPModel.h

```c++
/*
==========================================================================

Purpose:

	* 这个类CIOCPModel是本代码的核心类，用于说明WinSock服务器端编程模型中的
	  完成端口(IOCP)的使用方法，并使用MFC对话框程序来调用这个类实现了基本的
	  服务器网络通信的功能。

	* 其中的PER_IO_DATA结构体是封装了用于每一个重叠操作的参数
	  PER_HANDLE_DATA 是封装了用于每一个Socket的参数，也就是用于每一个完成端口的参数

Notes:

	* 具体讲明了服务器端建立完成端口、建立工作者线程、投递Recv请求、投递Accept请求的方法，
	  所有的客户端连入的Socket都需要绑定到IOCP上，所有从客户端发来的数据，都会实时显示到
	  主界面中去。

==========================================================================
*/

#pragma once

// winsock 2 的头文件和库
#include <winsock2.h>
#include <MSWSock.h>
#pragma comment(lib,"ws2_32.lib")

// 缓冲区长度 (1024*8)
// 之所以为什么设置8K，也是一个江湖上的经验值
// 如果确实客户端发来的每组数据都比较少，那么就设置得小一些，省内存
#define MAX_BUFFER_LEN        8192  
// 默认端口
#define DEFAULT_PORT          12345    
// 默认IP地址
#define DEFAULT_IP            _T("127.0.0.1")


//////////////////////////////////////////////////////////////////
// 在完成端口上投递的I/O操作的类型
typedef enum _OPERATION_TYPE  
{  
	ACCEPT_POSTED,                     // 标志投递的Accept操作
	SEND_POSTED,                       // 标志投递的是发送操作
	RECV_POSTED,                       // 标志投递的是接收操作
	NULL_POSTED                        // 用于初始化，无意义
} OPERATION_TYPE;

//====================================================================================
//
//				单IO数据结构体定义(用于每一个重叠操作的参数)
//
//====================================================================================

typedef struct _PER_IO_CONTEXT
{
	OVERLAPPED     m_Overlapped; // 每一个重叠网络操作的重叠结构(针对每一个Socket的每一个操作，都要有一个)              
	SOCKET         m_sockAccept; // 这个网络操作所使用的Socket
	WSABUF         m_wsaBuf; // WSA类型的缓冲区，用于给重叠操作传参数的
	char           m_szBuffer[MAX_BUFFER_LEN]; // 这个是WSABUF里具体存字符的缓冲区
	OPERATION_TYPE m_OpType; // 标识网络操作的类型(对应上面的枚举)

	// 初始化
	_PER_IO_CONTEXT()
	{
		ZeroMemory(&m_Overlapped, sizeof(m_Overlapped));  
		ZeroMemory(m_szBuffer, MAX_BUFFER_LEN);
		m_sockAccept = INVALID_SOCKET;
		m_wsaBuf.buf = m_szBuffer;
		m_wsaBuf.len = MAX_BUFFER_LEN;
		m_OpType     = NULL_POSTED;
	}
	// 释放掉Socket
	~_PER_IO_CONTEXT()
	{
		if (m_sockAccept != INVALID_SOCKET)
		{
			closesocket(m_sockAccept);
			m_sockAccept = INVALID_SOCKET;
		}
	}
	// 重置缓冲区内容
	void ResetBuffer()
	{
		ZeroMemory(m_szBuffer, MAX_BUFFER_LEN);
	}

} PER_IO_CONTEXT, *PPER_IO_CONTEXT;


//====================================================================================
//
//				单句柄数据结构体定义(用于每一个完成端口，也就是每一个Socket的参数)
//
//====================================================================================

typedef struct _PER_SOCKET_CONTEXT
{  
	SOCKET      m_Socket; // 每一个客户端连接的Socket
	SOCKADDR_IN m_ClientAddr; // 客户端的地址
	// CArray是MFC提供的一套模板库
	// 客户端网络操作的上下文数据，也就是说对于每一个客户端Socket，是可以在上面同时投递多个IO请求的
	CArray<_PER_IO_CONTEXT*> m_arrayIoContext;

	// 初始化
	_PER_SOCKET_CONTEXT()
	{
		m_Socket = INVALID_SOCKET;
		memset(&m_ClientAddr, 0, sizeof(m_ClientAddr)); 
	}

	// 释放资源
	~_PER_SOCKET_CONTEXT()
	{
		if (m_Socket != INVALID_SOCKET)
		{
			closesocket(m_Socket);
		    m_Socket = INVALID_SOCKET;
		}
		// 释放掉所有的IO上下文数据
		for (int i=0; i < m_arrayIoContext.GetCount(); i++)
		{
			delete m_arrayIoContext.GetAt(i);
		}
		m_arrayIoContext.RemoveAll();
	}

	// 获取一个新的IoContext
	_PER_IO_CONTEXT* GetNewIoContext()
	{
		_PER_IO_CONTEXT* p = new _PER_IO_CONTEXT;

		m_arrayIoContext.Add( p );

		return p;
	}

	// 从数组中移除一个指定的IoContext
	void RemoveContext( _PER_IO_CONTEXT* pContext )
	{
		ASSERT(pContext != NULL);

		for (int i = 0; i < m_arrayIoContext.GetCount(); i++)
		{
			if (pContext == m_arrayIoContext.GetAt(i))
			{
				delete pContext;
				pContext = NULL;
				m_arrayIoContext.RemoveAt(i);				
				break;
			}
		}
	}

} PER_SOCKET_CONTEXT, *PPER_SOCKET_CONTEXT;




//====================================================================================
//
//				CIOCPModel类定义
//
//====================================================================================

// 工作者线程的线程参数
class CIOCPModel;
typedef struct _tagThreadParams_WORKER
{
	CIOCPModel* pIOCPModel; // 类指针，用于调用类中的函数
	int         nThreadNo;  // 线程编号

} THREADPARAMS_WORKER, *PTHREADPARAM_WORKER; 

// CIOCPModel类
class CIOCPModel
{
public:
	CIOCPModel(void);
	~CIOCPModel(void);

public:

	// 启动服务器
	bool Start();

	//	停止服务器
	void Stop();

	// 加载Socket库
	bool LoadSocketLib();

	// 卸载Socket库，彻底完事
	void UnloadSocketLib() { WSACleanup(); }

	// 设置监听端口
	void SetPort( const int& nPort ) { m_nPort = nPort; }

	// 设置主界面的指针，用于调用显示信息到界面中
	void SetMainDlg( CDialog* p ) { m_pMain=p; }

protected:

	// 初始化IOCP
	bool _InitializeIOCP();

	// 初始化Socket
	bool _InitializeListenSocket();

	// 最后释放资源
	void _DeInitialize();

	// 投递Accept请求
	bool _PostAccept( PER_IO_CONTEXT* pAcceptIoContext ); 

	// 投递接收数据请求
	bool _PostRecv( PER_IO_CONTEXT* pIoContext );

	// 在有客户端连入的时候，进行处理
	bool _DoAccpet( PER_SOCKET_CONTEXT* pSocketContext, PER_IO_CONTEXT* pIoContext );

	// 在有接收的数据到达的时候，进行处理
	bool _DoRecv( PER_SOCKET_CONTEXT* pSocketContext, PER_IO_CONTEXT* pIoContext );

	// 将客户端的相关信息存储到数组中
	void _AddToContextList( PER_SOCKET_CONTEXT *pSocketContext );

	// 将客户端的信息从数组中移除
	void _RemoveContext( PER_SOCKET_CONTEXT *pSocketContext );

	// 清空客户端信息
	void _ClearContextList();

	// 将句柄绑定到完成端口中
	bool _AssociateWithIOCP( PER_SOCKET_CONTEXT *pContext);

	// 处理完成端口上的错误
	bool HandleError( PER_SOCKET_CONTEXT *pContext,const DWORD& dwErr );

	// 线程函数，为IOCP请求服务的工作者线程
	static DWORD WINAPI _WorkerThread(LPVOID lpParam);

	// 获得本机的处理器数量
	int _GetNoOfProcessors();

	// 判断客户端Socket是否已经断开
	bool _IsSocketAlive(SOCKET s);

	// 在主界面中显示信息
	void _ShowMessage( const CString szFormat,...) const;

private:

	HANDLE                       m_hShutdownEvent;     // 用来通知线程系统退出的事件，为了能够更好的退出线程

	HANDLE                       m_hIOCompletionPort;  // 完成端口的句柄

	HANDLE*                      m_phWorkerThreads;    // 工作者线程的句柄指针

	int		                     m_nThreads;           // 生成的线程数量

	CString                      m_strIP;              // 服务器端的IP地址
	int                          m_nPort;              // 服务器端的监听端口

	CDialog*                     m_pMain;              // 主界面的界面指针，用于在主界面中显示消息

	CRITICAL_SECTION             m_csContextList;      // 用于Worker线程同步的互斥量

	CArray<PER_SOCKET_CONTEXT*>  m_arrayClientContext; // 客户端Socket的Context信息        

	PER_SOCKET_CONTEXT*          m_pListenContext;     // 用于监听的Socket的Context信息

	LPFN_ACCEPTEX                m_lpfnAcceptEx;       // AcceptEx 和 GetAcceptExSockaddrs 的函数指针，用于调用这两个扩展函数
	LPFN_GETACCEPTEXSOCKADDRS    m_lpfnGetAcceptExSockAddrs; 

};
```

#### 2. 服务器IOCPModel.cpp

```c++
#include "StdAfx.h"
#include "IOCPModel.h"
#include "MainDlg.h"

// 每一个处理器上产生多少个线程(为了最大限度的提升服务器性能，详见配套文档)
#define WORKER_THREADS_PER_PROCESSOR 2
// 同时投递的Accept请求的数量(这个要根据实际的情况灵活设置)
#define MAX_POST_ACCEPT              10
// 传递给Worker线程的退出信号
#define EXIT_CODE                    NULL


// 释放指针和句柄资源的宏

// 释放指针宏
#define RELEASE(x)        {if(x != NULL ){delete x;x=NULL;}}
// 释放句柄宏
#define RELEASE_HANDLE(x) {if(x != NULL && x!=INVALID_HANDLE_VALUE){ CloseHandle(x);x = NULL;}}
// 释放Socket宏
#define RELEASE_SOCKET(x) {if(x !=INVALID_SOCKET) { closesocket(x);x=INVALID_SOCKET;}}



CIOCPModel::CIOCPModel(void):
							m_nThreads(0),
							m_hShutdownEvent(NULL),
							m_hIOCompletionPort(NULL),
							m_phWorkerThreads(NULL),
							m_strIP(DEFAULT_IP),
							m_nPort(DEFAULT_PORT),
							m_pMain(NULL),
							m_lpfnAcceptEx(NULL),
							m_pListenContext(NULL)
{
}


CIOCPModel::~CIOCPModel(void)
{
	// 确保资源彻底释放
	this->Stop();
}




///////////////////////////////////////////////////////////////////
// 工作者线程：  为IOCP请求服务的工作者线程
//         也就是每当完成端口上出现了完成数据包，就将之取出来进行处理的线程
///////////////////////////////////////////////////////////////////

DWORD WINAPI CIOCPModel::_WorkerThread(LPVOID lpParam)
{    
	THREADPARAMS_WORKER* pParam = (THREADPARAMS_WORKER*)lpParam;
	CIOCPModel* pIOCPModel = (CIOCPModel*)pParam->pIOCPModel;
	int nThreadNo = (int)pParam->nThreadNo;

	pIOCPModel->_ShowMessage("工作者线程启动，ID: %d.",nThreadNo);

	OVERLAPPED           *pOverlapped = NULL;
	PER_SOCKET_CONTEXT   *pSocketContext = NULL;
	DWORD                dwBytesTransfered = 0;

	// 循环处理请求，直到接收到Shutdown信息为止
	// WAIT_OBJECT_0表示有信号，参数2为0立即返回
	while (WAIT_OBJECT_0 != WaitForSingleObject(pIOCPModel->m_hShutdownEvent, 0))
	{
		BOOL bReturn = GetQueuedCompletionStatus(
			pIOCPModel->m_hIOCompletionPort,
			&dwBytesTransfered,
			(PULONG_PTR)&pSocketContext,
			&pOverlapped,
			INFINITE); // INFINITE表示一直挂起，直到有消息

		// 如果收到的是退出标志，则直接退出
		if ( EXIT_CODE == (DWORD)pSocketContext)
		{
			break;
		}

		// 判断是否出现了错误
		if( !bReturn )  
		{  
			DWORD dwErr = GetLastError();
			// 显示一下提示信息
			if (!pIOCPModel->HandleError(pSocketContext, dwErr))
			{
				break;
			}
			continue;
		}  
		else  
		{  	
			// 读取传入的参数
			PER_IO_CONTEXT* pIoContext = CONTAINING_RECORD(pOverlapped, PER_IO_CONTEXT, m_Overlapped);  

			// 判断是否有客户端断开了
			if((0 == dwBytesTransfered) && ( RECV_POSTED==pIoContext->m_OpType || SEND_POSTED==pIoContext->m_OpType))  
			{  
				pIOCPModel->_ShowMessage( _T("客户端 %s:%d 断开连接."),inet_ntoa(pSocketContext->m_ClientAddr.sin_addr), ntohs(pSocketContext->m_ClientAddr.sin_port) );

				// 释放掉对应的资源
				pIOCPModel->_RemoveContext( pSocketContext );

 				continue;  
			}  
			else
			{
				switch( pIoContext->m_OpType )  
				{  
					 // Accept  
				case ACCEPT_POSTED:
					{ 

						// 为了增加代码可读性，这里用专门的_DoAccept函数进行处理连入请求
						pIOCPModel->_DoAccpet( pSocketContext, pIoContext );						
						

					}
					break;

					// RECV
				case RECV_POSTED:
					{
						// 为了增加代码可读性，这里用专门的_DoRecv函数进行处理接收请求
						pIOCPModel->_DoRecv( pSocketContext,pIoContext );
					}
					break;

					// SEND
					// 这里略过不写了，要不代码太多了，不容易理解，Send操作相对来讲简单一些
				case SEND_POSTED:
					{

					}
					break;
				default:
					// 不应该执行到这里
					TRACE(_T("_WorkThread中的 pIoContext->m_OpType 参数异常.\n"));
					break;
				} //switch
			}//if
		}//if

	}//while

	TRACE(_T("工作者线程 %d 号退出.\n"), nThreadNo);
	// 释放线程参数
	RELEASE(lpParam);
	return 0;
}



//====================================================================================
//
//				    系统初始化和终止
//
//====================================================================================


////////////////////////////////////////////////////////////////////
// 初始化WinSock 2.2
bool CIOCPModel::LoadSocketLib()
{    
	WSADATA wsaData;
	int nResult = WSAStartup(MAKEWORD(2,2), &wsaData);
	// 错误(一般都不可能出现)
	if (NO_ERROR != nResult)
	{
		this->_ShowMessage(_T("初始化WinSock 2.2失败！\n"));
		return false; 
	}
	return true;
}

//////////////////////////////////////////////////////////////////
//	启动服务器
bool CIOCPModel::Start()
{
	// 初始化线程互斥量
	InitializeCriticalSection(&m_csContextList);

	// 建立系统退出的事件通知
	m_hShutdownEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

	// 初始化IOCP
	if (false == _InitializeIOCP())
	{
		this->_ShowMessage(_T("初始化IOCP失败！\n"));
		return false;
	}
	else
	{
		this->_ShowMessage("\nIOCP初始化完毕\n.");
	}

	// 初始化Socket
	if (false == _InitializeListenSocket())
	{
		this->_ShowMessage(_T("Listen Socket初始化失败！\n"));
		this->_DeInitialize();
		return false;
	}
	else
	{
		this->_ShowMessage("Listen Socket初始化完毕.");
	}

	this->_ShowMessage(_T("系统准备就绪，等候连接....\n"));

	return true;
}


////////////////////////////////////////////////////////////////////
//	开始发送系统退出消息，退出完成端口和线程资源
void CIOCPModel::Stop()
{
	if( m_pListenContext!=NULL && m_pListenContext->m_Socket!=INVALID_SOCKET )
	{
		// 激活关闭消息通知
		SetEvent(m_hShutdownEvent);

		for (int i = 0; i < m_nThreads; i++)
		{
			// 通知所有的完成端口操作退出
			PostQueuedCompletionStatus(m_hIOCompletionPort, 0, (DWORD)EXIT_CODE, NULL);
		}

		// 等待所有的客户端资源退出
		WaitForMultipleObjects(m_nThreads, m_phWorkerThreads, TRUE, INFINITE);

		// 清除客户端列表信息
		this->_ClearContextList();

		// 释放其他资源
		this->_DeInitialize();

		this->_ShowMessage("停止监听\n");
	}	
}


////////////////////////////////
// 初始化完成端口
bool CIOCPModel::_InitializeIOCP()
{
	// 建立第一个完成端口
	m_hIOCompletionPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

	if ( NULL == m_hIOCompletionPort)
	{
		this->_ShowMessage(_T("建立完成端口失败！错误代码: %d!\n"), WSAGetLastError());
		return false;
	}

	// 根据本机中的处理器数量，建立对应的线程数
	m_nThreads = WORKER_THREADS_PER_PROCESSOR * _GetNoOfProcessors();
	
	// 为工作者线程初始化句柄
	m_phWorkerThreads = new HANDLE[m_nThreads];
	
	// 根据计算出来的数量建立工作者线程
	DWORD nThreadID;
	for (int i = 0; i < m_nThreads; i++)
	{
		THREADPARAMS_WORKER* pThreadParams = new THREADPARAMS_WORKER;
		pThreadParams->pIOCPModel = this;
		pThreadParams->nThreadNo  = i + 1;
		m_phWorkerThreads[i] = ::CreateThread(0, 0, _WorkerThread, (void *)pThreadParams, 0, &nThreadID);
	}

	TRACE(" 建立 _WorkerThread %d 个.\n", m_nThreads );

	return true;
}


/////////////////////////////////////////////////////////////////
// 初始化Socket
bool CIOCPModel::_InitializeListenSocket()
{
	// AcceptEx 和 GetAcceptExSockaddrs 的GUID，用于导出函数指针
	GUID GuidAcceptEx = WSAID_ACCEPTEX;  
	GUID GuidGetAcceptExSockAddrs = WSAID_GETACCEPTEXSOCKADDRS; 

	// 服务器地址信息，用于绑定Socket
	struct sockaddr_in ServerAddress;

	// 生成用于监听的Socket的信息
	m_pListenContext = new PER_SOCKET_CONTEXT;

	// 需要使用重叠IO，必须得使用WSASocket来建立Socket，才可以支持重叠IO操作
	m_pListenContext->m_Socket = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
	if (INVALID_SOCKET == m_pListenContext->m_Socket) 
	{
		this->_ShowMessage("初始化Socket失败，错误代码: %d.\n", WSAGetLastError());
		return false;
	}
	else
	{
		TRACE("WSASocket() 完成.\n");
	}

	// 将Listen Socket绑定至完成端口中
	if( NULL== CreateIoCompletionPort( (HANDLE)m_pListenContext->m_Socket, m_hIOCompletionPort,(DWORD)m_pListenContext, 0))  
	{  
		this->_ShowMessage("绑定 Listen Socket至完成端口失败！错误代码: %d/n", WSAGetLastError());  
		RELEASE_SOCKET(m_pListenContext->m_Socket);
		return false;
	}
	else
	{
		TRACE("Listen Socket绑定完成端口 完成.\n");
	}

	// 填充地址信息
	ZeroMemory((char *)&ServerAddress, sizeof(ServerAddress));
	ServerAddress.sin_family = AF_INET;
	// 这里可以绑定任何可用的IP地址，或者绑定一个指定的IP地址 
	//ServerAddress.sin_addr.s_addr = htonl(INADDR_ANY);                      
	ServerAddress.sin_addr.s_addr = inet_addr(m_strIP.GetString());         
	ServerAddress.sin_port = htons(m_nPort);                          

	// 绑定地址和端口
	if (SOCKET_ERROR == bind(m_pListenContext->m_Socket, (struct sockaddr *) &ServerAddress, sizeof(ServerAddress))) 
	{
		this->_ShowMessage("bind()函数执行错误.\n");
		return false;
	}
	else
	{
		TRACE("bind() 完成.\n");
	}

	// 开始进行监听
	if (SOCKET_ERROR == listen(m_pListenContext->m_Socket, SOMAXCONN))
	{
		this->_ShowMessage("Listen()函数执行出现错误.\n");
		return false;
	}
	else
	{
		TRACE("Listen() 完成.\n");
	}

	// 使用AcceptEx函数，因为这个是属于WinSock2规范之外的微软另外提供的扩展函数
	// 所以需要额外获取一下函数的指针，
	// 获取AcceptEx函数指针
	DWORD dwBytes = 0;  
	if(SOCKET_ERROR == WSAIoctl(
		m_pListenContext->m_Socket, 
		SIO_GET_EXTENSION_FUNCTION_POINTER, 
		&GuidAcceptEx, 
		sizeof(GuidAcceptEx), 
		&m_lpfnAcceptEx, 
		sizeof(m_lpfnAcceptEx), 
		&dwBytes, 
		NULL, 
		NULL))  
	{  
		this->_ShowMessage("WSAIoctl 未能获取AcceptEx函数指针。错误代码: %d\n", WSAGetLastError()); 
		this->_DeInitialize();
		return false;
	}  

	// 获取GetAcceptExSockAddrs函数指针，也是同理
	if(SOCKET_ERROR == WSAIoctl(
		m_pListenContext->m_Socket, 
		SIO_GET_EXTENSION_FUNCTION_POINTER, 
		&GuidGetAcceptExSockAddrs,
		sizeof(GuidGetAcceptExSockAddrs), 
		&m_lpfnGetAcceptExSockAddrs, 
		sizeof(m_lpfnGetAcceptExSockAddrs),   
		&dwBytes, 
		NULL, 
		NULL))  
	{  
		this->_ShowMessage("WSAIoctl 未能获取GuidGetAcceptExSockAddrs函数指针。错误代码: %d\n", WSAGetLastError());  
		this->_DeInitialize();
		return false; 
	}  


	// 为AcceptEx 准备参数，然后投递AcceptEx I/O请求
	for (int i=0; i < MAX_POST_ACCEPT; i++)
	{
		// 新建一个IO_CONTEXT
		PER_IO_CONTEXT* pAcceptIoContext = m_pListenContext->GetNewIoContext();

		if (false == this->_PostAccept(pAcceptIoContext))
		{
			m_pListenContext->RemoveContext(pAcceptIoContext);
			return false;
		}
	}

	this->_ShowMessage( _T("投递 %d 个AcceptEx请求完毕"), MAX_POST_ACCEPT);
	return true;
}

////////////////////////////////////////////////////////////
//	最后释放掉所有资源
void CIOCPModel::_DeInitialize()
{
	// 删除客户端列表的互斥量
	DeleteCriticalSection(&m_csContextList);

	// 关闭系统退出事件句柄
	RELEASE_HANDLE(m_hShutdownEvent);

	// 释放工作者线程句柄指针
	for( int i=0;i<m_nThreads;i++ )
	{
		RELEASE_HANDLE(m_phWorkerThreads[i]);
	}
	
	RELEASE(m_phWorkerThreads);

	// 关闭IOCP句柄
	RELEASE_HANDLE(m_hIOCompletionPort);

	// 关闭监听Socket
	RELEASE(m_pListenContext);

	this->_ShowMessage("释放资源完毕.\n");
}


//====================================================================================
//
//				    投递完成端口请求
//
//====================================================================================


//////////////////////////////////////////////////////////////////
// 投递Accept请求
bool CIOCPModel::_PostAccept( PER_IO_CONTEXT* pAcceptIoContext )
{
	ASSERT(INVALID_SOCKET != m_pListenContext->m_Socket );

	// 准备参数
	DWORD dwBytes = 0;  
	pAcceptIoContext->m_OpType = ACCEPT_POSTED;  
	WSABUF *p_wbuf   = &pAcceptIoContext->m_wsaBuf;
	OVERLAPPED *p_ol = &pAcceptIoContext->m_Overlapped;
	
	// 为以后新连入的客户端先准备好Socket( 这个是与传统accept最大的区别 ) 
	pAcceptIoContext->m_sockAccept  = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, WSA_FLAG_OVERLAPPED);  
	if( INVALID_SOCKET == pAcceptIoContext->m_sockAccept )  
	{  
		_ShowMessage("创建用于Accept的Socket失败！错误代码: %d", WSAGetLastError()); 
		return false;
	} 

	// 投递AcceptEx
	BOOL ret = m_lpfnAcceptEx(m_pListenContext->m_Socket, pAcceptIoContext->m_sockAccept, p_wbuf->buf, p_wbuf->len - ((sizeof(SOCKADDR_IN) + 16) * 2),
		sizeof(SOCKADDR_IN) + 16, sizeof(SOCKADDR_IN) + 16, &dwBytes, p_ol);
	if (!ret && WSA_IO_PENDING != WSAGetLastError())
	{  
		_ShowMessage("投递 AcceptEx 请求失败，错误代码: %d", WSAGetLastError());
		return false;
	} 

	return true;
}

////////////////////////////////////////////////////////////
// 在有客户端连入的时候，进行处理
// 流程有点复杂，你要是看不懂的话，就看配套的文档吧....
// 如果能理解这里的话，完成端口的机制你就消化了一大半了

// 总之你要知道，传入的是ListenSocket的Context，我们需要复制一份出来给新连入的Socket用
// 原来的Context还是要在上面继续投递下一个Accept请求
//
bool CIOCPModel::_DoAccpet( PER_SOCKET_CONTEXT* pSocketContext, PER_IO_CONTEXT* pIoContext )
{
	SOCKADDR_IN* ClientAddr = NULL;
	SOCKADDR_IN* LocalAddr = NULL;  
	int remoteLen = sizeof(SOCKADDR_IN), localLen = sizeof(SOCKADDR_IN);  

	///////////////////////////////////////////////////////////////////////////
	// 1. 首先取得连入客户端的地址信息
	// 这个 m_lpfnGetAcceptExSockAddrs 不得了啊~~~~~~
	// 不但可以取得客户端和本地端的地址信息，还能顺便取出客户端发来的第一组数据，老强大了...
	this->m_lpfnGetAcceptExSockAddrs(pIoContext->m_wsaBuf.buf, pIoContext->m_wsaBuf.len - ((sizeof(SOCKADDR_IN)+16)*2),  
		sizeof(SOCKADDR_IN)+16, sizeof(SOCKADDR_IN)+16, (LPSOCKADDR*)&LocalAddr, &localLen, (LPSOCKADDR*)&ClientAddr, &remoteLen);  

	this->_ShowMessage( _T("客户端 %s:%d 连入."), inet_ntoa(ClientAddr->sin_addr), ntohs(ClientAddr->sin_port) );
	this->_ShowMessage( _T("客户额 %s:%d 信息：%s."),inet_ntoa(ClientAddr->sin_addr), ntohs(ClientAddr->sin_port),pIoContext->m_wsaBuf.buf );


	//////////////////////////////////////////////////////////////////////////////////////////////////////
	// 2. 这里需要注意，这里传入的这个是ListenSocket上的Context，这个Context我们还需要用于监听下一个连接
	// 所以我还得要将ListenSocket上的Context复制出来一份为新连入的Socket新建一个SocketContext

	PER_SOCKET_CONTEXT* pNewSocketContext = new PER_SOCKET_CONTEXT;
	pNewSocketContext->m_Socket           = pIoContext->m_sockAccept;
	memcpy(&(pNewSocketContext->m_ClientAddr), ClientAddr, sizeof(SOCKADDR_IN));

	// 参数设置完毕，将这个Socket和完成端口绑定(这也是一个关键步骤)
	if (false == this->_AssociateWithIOCP(pNewSocketContext))
	{
		RELEASE( pNewSocketContext );
		return false;
	}  


	///////////////////////////////////////////////////////////////////////////////////////////////////
	// 3. 继续，建立其下的IoContext，用于在这个Socket上投递第一个Recv数据请求
	PER_IO_CONTEXT* pNewIoContext = pNewSocketContext->GetNewIoContext();
	pNewIoContext->m_OpType       = RECV_POSTED;
	pNewIoContext->m_sockAccept   = pNewSocketContext->m_Socket;
	// 如果Buffer需要保留，就自己拷贝一份出来
	//memcpy( pNewIoContext->m_szBuffer,pIoContext->m_szBuffer,MAX_BUFFER_LEN );

	// 绑定完毕之后，就可以开始在这个Socket上投递完成请求了
	if (false == this->_PostRecv(pNewIoContext))
	{
		pNewSocketContext->RemoveContext(pNewIoContext);
		return false;
	}

	/////////////////////////////////////////////////////////////////////////////////////////////////
	// 4. 如果投递成功，那么就把这个有效的客户端信息，加入到ContextList中去(需要统一管理，方便释放资源)
	this->_AddToContextList(pNewSocketContext);

	////////////////////////////////////////////////////////////////////////////////////////////////
	// 5. 使用完毕之后，把Listen Socket的那个IoContext重置，然后准备投递新的AcceptEx
	pIoContext->ResetBuffer();
	return this->_PostAccept(pIoContext);	
}

////////////////////////////////////////////////////////////////////
// 投递接收数据请求
bool CIOCPModel::_PostRecv( PER_IO_CONTEXT* pIoContext )
{
	// 初始化变量
	DWORD dwFlags = 0;
	DWORD dwBytes = 0;
	WSABUF *p_wbuf   = &pIoContext->m_wsaBuf;
	OVERLAPPED *p_ol = &pIoContext->m_Overlapped;

	pIoContext->ResetBuffer();
	pIoContext->m_OpType = RECV_POSTED;

	// 初始化完成后，，投递WSARecv请求
	int nBytesRecv = WSARecv( pIoContext->m_sockAccept, p_wbuf, 1, &dwBytes, &dwFlags, p_ol, NULL );

	// 如果返回值错误，并且错误的代码并非是Pending的话，那就说明这个重叠请求失败了
	if ((SOCKET_ERROR == nBytesRecv) && (WSA_IO_PENDING != WSAGetLastError()))
	{
		this->_ShowMessage("投递第一个WSARecv失败！");
		return false;
	}
	return true;
}

/////////////////////////////////////////////////////////////////
// 在有接收的数据到达的时候，进行处理
bool CIOCPModel::_DoRecv( PER_SOCKET_CONTEXT* pSocketContext, PER_IO_CONTEXT* pIoContext )
{
	// 先把上一次的数据显示出现，然后就重置状态，发出下一个Recv请求
	SOCKADDR_IN* ClientAddr = &pSocketContext->m_ClientAddr;
	this->_ShowMessage( _T("收到  %s:%d 信息：%s"),inet_ntoa(ClientAddr->sin_addr), ntohs(ClientAddr->sin_port),pIoContext->m_wsaBuf.buf );

	// 然后开始投递下一个WSARecv请求
	return _PostRecv( pIoContext );
}



/////////////////////////////////////////////////////
// 将句柄(Socket)绑定到完成端口中
bool CIOCPModel::_AssociateWithIOCP( PER_SOCKET_CONTEXT *pContext )
{
	// 将用于和客户端通信的SOCKET绑定到完成端口中
	HANDLE hTemp = CreateIoCompletionPort((HANDLE)pContext->m_Socket, m_hIOCompletionPort, (DWORD)pContext, 0);

	if (NULL == hTemp)
	{
		this->_ShowMessage(("执行CreateIoCompletionPort()出现错误.错误代码：%d"),GetLastError());
		return false;
	}

	return true;
}




//====================================================================================
//
//				    ContextList 相关操作
//
//====================================================================================


//////////////////////////////////////////////////////////////
// 将客户端的相关信息存储到数组中
void CIOCPModel::_AddToContextList( PER_SOCKET_CONTEXT *pHandleData )
{
	EnterCriticalSection(&m_csContextList);

	m_arrayClientContext.Add(pHandleData);	
	
	LeaveCriticalSection(&m_csContextList);
}

////////////////////////////////////////////////////////////////
//	移除某个特定的Context
void CIOCPModel::_RemoveContext( PER_SOCKET_CONTEXT *pSocketContext )
{
	EnterCriticalSection(&m_csContextList);

	for (int i=0; i < m_arrayClientContext.GetCount(); i++ )
	{
		if (pSocketContext == m_arrayClientContext.GetAt(i) )
		{
			RELEASE( pSocketContext );			
			m_arrayClientContext.RemoveAt(i);			
			break;
		}
	}

	LeaveCriticalSection(&m_csContextList);
}

////////////////////////////////////////////////////////////////
// 清空客户端信息
void CIOCPModel::_ClearContextList()
{
	EnterCriticalSection(&m_csContextList);

	for( int i=0;i<m_arrayClientContext.GetCount();i++ )
	{
		delete m_arrayClientContext.GetAt(i);
	}

	m_arrayClientContext.RemoveAll();

	LeaveCriticalSection(&m_csContextList);
}



//====================================================================================
//
//				       其他辅助函数定义
//
//====================================================================================


///////////////////////////////////////////////////////////////////
// 获得本机中处理器的数量
int CIOCPModel::_GetNoOfProcessors()
{
	SYSTEM_INFO si;

	GetSystemInfo(&si);

	return si.dwNumberOfProcessors;
}

/////////////////////////////////////////////////////////////////////
// 在主界面中显示提示信息
void CIOCPModel::_ShowMessage(const CString szFormat,...) const
{
	// 根据传入的参数格式化字符串
	CString   strMessage;
	va_list   arglist;

	// 处理变长参数
	va_start(arglist, szFormat);
	strMessage.FormatV(szFormat, arglist);
	va_end(arglist);

	// 在主界面中显示
	CMainDlg* pMain = (CMainDlg*)m_pMain;
	if (m_pMain != NULL)
	{
		pMain->AddInformation(strMessage);
		TRACE( strMessage+_T("\n") );
	}	
}

/////////////////////////////////////////////////////////////////////
// 判断客户端Socket是否已经断开，否则在一个无效的Socket上投递WSARecv操作会出现异常
// 使用的方法是尝试向这个socket发送数据，判断这个socket调用的返回值
// 因为如果客户端网络异常断开(例如客户端崩溃或者拔掉网线等)的时候，服务器端是无法收到客户端断开的通知的

bool CIOCPModel::_IsSocketAlive(SOCKET s)
{
	int nByteSent = send(s, "", 0, 0);
	if (-1 == nByteSent) return false;
	return true;
}

///////////////////////////////////////////////////////////////////
// 显示并处理完成端口上的错误
bool CIOCPModel::HandleError( PER_SOCKET_CONTEXT *pContext,const DWORD& dwErr )
{
	// 如果是超时了，就再继续等吧  
	if(WAIT_TIMEOUT == dwErr)  
	{  	
		// 确认客户端是否还活着...
		if( !_IsSocketAlive( pContext->m_Socket) )
		{
			this->_ShowMessage( _T("检测到客户端异常退出！") );
			this->_RemoveContext(pContext);
			return true;
		}
		else
		{
			this->_ShowMessage( _T("网络操作超时！重试中...") );
			return true;
		}
	}
	// 可能是客户端异常退出了
	else if(ERROR_NETNAME_DELETED == dwErr)
	{
		this->_ShowMessage( _T("检测到客户端异常退出！") );
		this->_RemoveContext(pContext);
		return true;
	}
	else
	{
		this->_ShowMessage( _T("完成端口操作出现错误，线程退出。错误代码：%d"), dwErr);
		return false;
	}
}
```

#### 3. 客户端Client.h

```c++
/*
==========================================================================

Purpose:

* 这个类CClient是本代码的核心类，用于产生用于指定的并发线程向指定服务器发送
  信息，测试服务器的响应及资源占用率情况，并使用了MFC对话框程序来进行说明

Notes:

* 客户端使用的是最简单的多线程阻塞式Socket，而且每个线程只发送一次数据
  如果需要可以修改成发送多次数据的情况

==========================================================================
*/

#pragma once

// 屏蔽deprecation警告
#pragma warning(disable: 4996)

// 缓冲区长度(8*1024字节)
#define MAX_BUFFER_LEN 8196    
#define DEFAULT_PORT          12345                      // 默认端口
#define DEFAULT_IP            _T("127.0.0.1")            // 默认IP地址
#define DEFAULT_THREADS       100                        // 默认并发线程数
#define DEFAULT_MESSAGE       _T("Hello!")   // 默认的发送信息

class CClient;

// 用于发送数据的线程参数
typedef struct _tagThreadParams_WORKER
{
	CClient* pClient;                               // 类指针，用于调用类中的函数
	SOCKET   sock;                                  // 每个线程使用的Socket
	int      nThreadNo;                             // 线程编号
	char     szBuffer[MAX_BUFFER_LEN];

} THREADPARAMS_WORKER,*PTHREADPARAM_WORKER;  

// 产生Socket连接的线程
typedef struct _tagThreadParams_CONNECTION
{
	CClient* pClient;                               // 类指针，用于调用类中的函数

} THREADPARAMS_CONNECTION,*PTHREADPARAM_CONNECTION; 


class CClient
{
public:
	CClient(void);
	~CClient(void);

public:

	// 加载Socket库
	bool LoadSocketLib();
	// 卸载Socket库，彻底完事
	void UnloadSocketLib() { WSACleanup(); }

	// 开始测试
	bool Start();
	//	停止测试
	void Stop();

	// 设置连接IP地址
	void SetIP( const CString& strIP ) { m_strServerIP=strIP; }
	// 设置监听端口
	void SetPort( const int& nPort )   { m_nPort=nPort; }
	// 设置并发线程数量
	void SetThreads(const int& n)      { m_nThreads=n; }
	// 设置要按发送的信息
	void SetMessage( const CString& strMessage ) { m_strMessage=strMessage; }

	// 设置主界面的指针，用于调用其函数
	void SetMainDlg( CDialog* p )      { m_pMain=p; }

	// 在主界面中显示信息
	void ShowMessage( const CString strInfo,...);

private:

	// 建立连接
	bool EstablishConnections();
	// 向服务器进行连接
	bool ConnetToServer( SOCKET *pSocket, CString strServer, int nPort );
	// 用于建立连接的线程
	static DWORD WINAPI _ConnectionThread(LPVOID lpParam);
	// 用于发送信息的线程
	static DWORD WINAPI _WorkerThread(LPVOID lpParam);

	// 释放资源
	void CleanUp();

private:

	CDialog*  m_pMain;                                      // 界面指针

	CString   m_strServerIP;                                // 服务器端的IP地址
	CString   m_strLocalIP;                                 // 本机IP地址
	CString   m_strMessage;                                 // 发给服务器的信息
	int       m_nPort;                                      // 监听端口
	int       m_nThreads;                                   // 并发线程数量

	HANDLE    *m_phWorkerThreads;
	HANDLE    m_hConnectionThread;                          // 接受连接的线程句柄
	HANDLE    m_hShutdownEvent;                             // 用来通知线程系统退出的事件，为了能够更好的退出线程

	THREADPARAMS_WORKER      *m_pParamsWorker;              // 线程参数
};
```

#### 4. 客户端Client.cpp

```c++
#include "StdAfx.h"
#include "Client.h"
#include "MainDlg.h"

#include <winsock2.h>

#pragma comment(lib,"ws2_32.lib")

#define RELEASE_HANDLE(x)               {if(x != NULL && x!=INVALID_HANDLE_VALUE){ CloseHandle(x);x = NULL;}}
#define RELEASE(x)                      {if(x != NULL ){delete x;x=NULL;}}

CClient::CClient(void):
			m_strServerIP(DEFAULT_IP),
			m_strLocalIP(DEFAULT_IP),
			m_nThreads(DEFAULT_THREADS),
			m_pMain(NULL),
			m_nPort(DEFAULT_PORT),
			m_strMessage(DEFAULT_MESSAGE),
			m_phWorkerThreads(NULL),
			m_hConnectionThread(NULL),
			m_hShutdownEvent(NULL)

{
}

CClient::~CClient(void)
{
	this->Stop();
}

//////////////////////////////////////////////////////////////////////////////////
//	建立连接的线程
DWORD WINAPI CClient::_ConnectionThread(LPVOID lpParam)
{
	THREADPARAMS_CONNECTION* pParams = (THREADPARAMS_CONNECTION*) lpParam;
	CClient* pClient = (CClient*)pParams->pClient;

	TRACE("_AccpetThread启动，系统监听中...\n");

	pClient->EstablishConnections();

	TRACE(_T("_ConnectionThread线程结束.\n"));

	RELEASE(pParams);	

	return 0;
}

/////////////////////////////////////////////////////////////////////////////////
// 用于发送信息的线程
DWORD WINAPI CClient::_WorkerThread(LPVOID lpParam)
{
	THREADPARAMS_WORKER *pParams = (THREADPARAMS_WORKER *)lpParam;
	CClient* pClient = (CClient*) pParams->pClient;

	char szTemp[MAX_BUFFER_LEN];
	memset( szTemp,0,sizeof(szTemp) );
	char szRecv[MAX_BUFFER_LEN];
	memset(szRecv,0,MAX_BUFFER_LEN);

	int nBytesSent = 0;
	int nBytesRecv = 0;

	//CopyMemory(szTemp,pParams->szBuffer,sizeof(pParams->szBuffer));

	// 向服务器发送信息
	sprintf( szTemp,("第1条信息：%s"),pParams->szBuffer );
	nBytesSent = send(pParams->sock, szTemp, strlen(szTemp), 0);
	if (SOCKET_ERROR == nBytesSent) 
	{
		TRACE("错误：发送1次信息失败，错误代码：%ld\n", WSAGetLastError());
		return 1; 
	}	
	TRACE("向服务器发送信息成功: %s\n", szTemp);
	pClient->ShowMessage("向服务器发送信息成功: %s", szTemp);

	Sleep( 3000 );

	// 再发送一条信息
	memset( szTemp,0,sizeof(szTemp) );
	sprintf( szTemp,("第2条信息：%s"),pParams->szBuffer );
	nBytesSent = send(pParams->sock, szTemp, strlen(szTemp), 0);
	if (SOCKET_ERROR == nBytesSent) 
	{
		TRACE("错误：发送第2次信息失败，错误代码：%ld\n", WSAGetLastError());
		return 1; 
	}	
	
	TRACE("向服务器发送信息成功: %s\n", szTemp);
	pClient->ShowMessage("向服务器发送信息成功: %s", szTemp);

	Sleep( 3000 );
	
	// 发第3条信息
	memset( szTemp,0,sizeof(szTemp) );
	sprintf( szTemp,("第3条信息：%s"),pParams->szBuffer );
	nBytesSent = send(pParams->sock, szTemp, strlen(szTemp), 0);
	if (SOCKET_ERROR == nBytesSent) 
	{
		TRACE("错误：发送第3次信息失败，错误代码：%ld\n", WSAGetLastError());
		return 1; 
	}	

	TRACE("向服务器发送信息成功: %s\n", szTemp);
	pClient->ShowMessage("向服务器发送信息成功: %s", szTemp);

	if (pParams->nThreadNo == pClient->m_nThreads)
	{
		pClient->ShowMessage(_T("测试并发 %d 个线程完毕."), pClient->m_nThreads);
	}

	return 0;
}
///////////////////////////////////////////////////////////////////////////////////
// 建立连接
bool  CClient::EstablishConnections()
{
	DWORD nThreadID;

	m_phWorkerThreads = new HANDLE[m_nThreads];
	m_pParamsWorker = new THREADPARAMS_WORKER[m_nThreads];

	// 根据用户设置的线程数量，生成每一个线程连接至服务器，并生成线程发送数据
	for (int i=0; i<m_nThreads; i++)
	{
		// 监听用户的停止事件
		if(WAIT_OBJECT_0 == WaitForSingleObject(m_hShutdownEvent, 0))
		{
			TRACE(_T("接收到用户停止命令.\n"));
			return true;
		}
		
		// 向服务器进行连接
		if( !this->ConnetToServer(&m_pParamsWorker[i].sock, m_strServerIP, m_nPort) )
		{
			ShowMessage(_T("连接服务器失败！"));
			CleanUp();
			return false;
		}

		m_pParamsWorker[i].nThreadNo = i+1;
		sprintf(m_pParamsWorker[i].szBuffer, "%d号线程 发送数据 %s", i+1, m_strMessage.GetString() );

		Sleep(10);

		// 如果连接服务器成功，就开始建立工作者线程，向服务器发送指定数据
		m_pParamsWorker[i].pClient = this;
		m_phWorkerThreads[i] = CreateThread(0, 0, _WorkerThread, (void *)(&m_pParamsWorker[i]), 0, &nThreadID);
	}

	return true;
}

////////////////////////////////////////////////////////////////////////////////////
//	向服务器进行Socket连接
bool CClient::ConnetToServer( SOCKET *pSocket, CString strServer, int nPort )
{
	struct sockaddr_in ServerAddress;
	struct hostent *Server;

	// 生成SOCKET
	*pSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

	if (INVALID_SOCKET == *pSocket) 
	{
		TRACE("错误：初始化Socket失败，错误信息：%d\n", WSAGetLastError());
		return false;
	}

	// 生成地址信息
	Server = gethostbyname(strServer.GetString());
	if (Server == NULL) 
	{
		closesocket(*pSocket);
		TRACE("错误：无效的服务器地址.\n");
		return false; 
	}

	
	ZeroMemory((char *) &ServerAddress, sizeof(ServerAddress));
	ServerAddress.sin_family = AF_INET;
	CopyMemory((char *)&ServerAddress.sin_addr.s_addr, 
		       (char *)Server->h_addr,
		        Server->h_length);

	ServerAddress.sin_port = htons(m_nPort);

	// 开始连接服务器
	if (SOCKET_ERROR == connect(*pSocket, reinterpret_cast<const struct sockaddr *>(&ServerAddress),sizeof(ServerAddress))) 
	{
		closesocket(*pSocket);
		TRACE("错误：连接至服务器失败！\n");
		return false; 
	}

	return true;
}

////////////////////////////////////////////////////////////////////
// 初始化WinSock 2.2
bool CClient::LoadSocketLib()
{    
	WSADATA wsaData;
	int nResult = WSAStartup(MAKEWORD(2,2), &wsaData);

	if (NO_ERROR != nResult)
	{
		ShowMessage(_T("初始化WinSock 2.2失败！\n"));
		return false; // 错误
	}

	return true;
}

///////////////////////////////////////////////////////////////////
// 开始监听
bool CClient::Start()
{
	// 建立系统退出的事件通知
	m_hShutdownEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

	// 启动连接线程
	DWORD nThreadID;
	THREADPARAMS_CONNECTION* pThreadParams = new THREADPARAMS_CONNECTION;
	pThreadParams->pClient = this;
	m_hConnectionThread = ::CreateThread(0, 0, _ConnectionThread, (void *)pThreadParams, 0, &nThreadID);

	return true;
}

///////////////////////////////////////////////////////////////////////
//	停止监听
void CClient::Stop()
{
	if( m_hShutdownEvent==NULL ) return ;

	SetEvent(m_hShutdownEvent);
	// 等待Connection线程退出
	WaitForSingleObject(m_hConnectionThread, INFINITE);

	// 关闭所有的Socket
	for (int i= 0; i< m_nThreads; i++)
	{
		closesocket(m_pParamsWorker[i].sock);
	}

	// 等待所有的工作者线程退出
	WaitForMultipleObjects(m_nThreads, m_phWorkerThreads, TRUE, INFINITE);

	// 清空资源
	CleanUp();

	TRACE("测试停止.\n");
}

//////////////////////////////////////////////////////////////////////
//	清空资源
void CClient::CleanUp()
{
	if(m_hShutdownEvent==NULL) return;

	RELEASE(m_phWorkerThreads);

	RELEASE_HANDLE(m_hConnectionThread);

	RELEASE(m_pParamsWorker);

	RELEASE_HANDLE(m_hShutdownEvent);
}

/////////////////////////////////////////////////////////////////////
// 在主界面中显示信息
void CClient::ShowMessage(const CString strInfo,...)
{
	// 根据传入的参数格式化字符串
	CString   strMessage;
	va_list   arglist;

	va_start(arglist, strInfo);
	strMessage.FormatV(strInfo,arglist);
	va_end(arglist);

	// 在主界面中显示
	CMainDlg* pMain = (CMainDlg*)m_pMain;
	if( m_pMain!=NULL )
	{
		pMain->AddInformation(strMessage);
	}
}
```

# 二. Linux系统

linux io系统调用发展历程
(同步接口：)

➔ read(2)/write(2)

➔ pread(2)、readv(2)、preadv(2)、preadv2(2)

(异步接口：)

➔ aio_read(2)/aio_write(2)

➔ io_uring since Linux Kernel 5.1

## 2.1 select

## 2.2 poll

## 2.3 epoll

### 1. epoll接口概述

epoll是linux中IO多路复用的一种机制，I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。当然linux中IO多路复用不仅仅是epoll，其他多路复用机制还有select、poll。

1. 创建

   ```c
   #include <sys/epoll.h>
   
   // size用来告诉内核这个监听的数目一共有多大，已废弃大于0即可
   int epoll_create(int size);
   // 如果flags为0，那么除了删除过时的size参数之外，epoll_create1()与epoll_create()相同。
   // 标志EPOLL_CLOEXEC：在新的文件描述符上设置“执行时关闭”(FD_CLOEXEC)标志。参见open(2)中对O_CLOEXEC标志的描述。
   int epoll_create1(int flags);
   ```

   创建一个epoll的句柄。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

   ```shell
   # 查看注册到epoll中文件描述符的限制
   $ cat /proc/sys/fs/epoll/max_user_watches
   1657221
   ```

2. `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);`
   epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。调用成功，返回0；调用失败，返回-1。kcmp(2) KCMP_EPOLL_TFD操作可用于测试epoll实例中是否存在文件描述符。

   第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：
   EPOLL_CTL_ADD：注册新的fd到epfd中；
   EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
   EPOLL_CTL_DEL：从epfd中删除一个fd；
   第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

```text
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

events可以是以下几个宏的集合：

| 宏             | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| EPOLLIN        | 数据可读（对端SOCKET正常关闭会放入文件结束符）               |
| EPOLLPRI       | 文件描述符上有一些异常条件。可能包括:<br/>•TCP套接字上有带外数据(参见TCP(7))。<br/>•在包模式下的伪终端主机在从机上看到了状态变化(见ioctl_tty(2))。<br/>•cgroup。事件文件已被修改(参见cgroups(7))。 |
| EPOLLOUT       | 写入现在是可能的                                             |
| EPOLLERR       | 在关联的文件描述符上发生错误。当管道的读端关闭时，也会为管道的写端报告此事件。<br/>epoll_wait(2)将始终报告此事件;在调用epoll_ctl()时，没有必要在事件中设置它。 |
| EPOLLHUP       | 关联文件描述符上发生挂起。<br/>epoll_wait(2)将一直等待此事件;在调用epoll_ctl()时，没有必要在事件中设置它。<br/>注意，当从管道或流套接字等通道读取时，此事件仅表示对等端关闭了通道的一端。只有在使用了通道中所有未完成的数据之后，从通道进行的后续读取才会返回0(文件结束)。 |
| EPOLLNVAL      | fd没有打开                                                   |
| EPOLLRDNORM    | 普通数据可读                                                 |
| EPOLLRDBAND    | 优先级带数据可读                                             |
| EPOLLWRNORM    | 普通数据可写                                                 |
| EPOLLWRBAND    | 优先级带数据可写                                             |
| EPOLLMSG       |                                                              |
| EPOLLRDHUP     | TCP连接对方关闭或者对方关闭了写操作。当使用边缘触发监视时，这个标志对于编写检测对等关闭的简单代码特别有用。 |
| EPOLLEXCLUSIVE | 为附加到目标文件描述符fd的epoll文件描述符设置独占唤醒模式。当唤醒事件发生，多个epoll文件描述符使用EPOLLEXCLUSIVE附加到同一个目标文件时，一个或多个epoll文件描述符将使用epoll_wait(2)接收一个事件。在此场景中(当EPOLLEXCLUSIVE未设置时)，默认设置是让所有epoll文件描述符接收事件。因此，EPOLLEXCLUSIVE在某些情况下有助于避免雷电羊群问题。<br>如果同一个文件描述符存在于多个epoll实例中，其中一些带有EPOLLEXCLUSIVE标志，而另一些没有，那么事件将被提供给所有没有指定EPOLLEXCLUSIVE的epoll实例，以及至少一个指定了EPOLLEXCLUSIVE的epoll实例。<br>以下值可以与EPOLLEXCLUSIVE一起指定:EPOLLIN、EPOLLOUT、EPOLLWAKEUP和EPOLLET。还可以指定EPOLLHUP和EPOLLERR，但这不是必需的:通常情况下，如果发生了这些事件，无论它们是否在事件中指定，都会报告它们。试图在事件中指定其他值会导致错误EINVAL。<br/>EPOLLEXCLUSIVE只能在EPOLL_CTL_ADD操作中使用;尝试将它与EPOLL_CTL_MOD一起使用会产生错误。如果已经使用epoll_ctl()设置了EPOLLEXCLUSIVE，那么同一个epfd, fd对上的后续EPOLL_CTL_MOD将产生错误。在事件中指定EPOLLEXCLUSIVE并将目标文件描述符fd指定为epoll实例的epoll_ctl()调用同样会失败。所有这些情况下的错误都是EINVAL。<br/>EPOLLEXCLUSIVE标志是事件的输入标志。当调用epoll_ctl()时，事件字段永远不会被epoll_wait(2)返回。 |
| EPOLLWAKEUP    | 请求处理系统唤醒事件，以防止在处理这些事件时发生系统挂起。<br/>如果系统通过/sys/power/autosleep处于自动休眠模式，并且发生了一个事件将设备从休眠状态中唤醒，设备驱动程序将保持设备处于休眠状态，直到该事件排队。为了使设备在事件被处理之前保持唤醒，必须使用epoll_ctl(2) EPOLLWAKEUP标志。<br>当在结构epoll_event的events字段中设置了EPOLLWAKEUP标志时，系统将从事件排队的那一刻起，通过返回事件的epoll_wait(2)调用一直保持清醒，直到后续的epoll_wait(2)调用。如果该事件使系统在该时间之后仍处于唤醒状态，则应该在第二次epoll_wait(2)调用之前执行一个单独的wake_lock。<br>假设既没有设置EPOLLET也没有设置EPOLLONESHOT，在使用唤醒事件后再次调用epoll_wait之前，不会重新允许系统挂起。需要CAP_BLOCK_SUSPEND。 |
| EPOLLONESHOT   | 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到epoll队列里。 |
| EPOLLET        | 将EPOLL设为边缘触发(Edge Triggered)模式，默认水平触发(Level Triggered)。<br>水平触发(level-triggered)：<br>socket接收缓冲区不为空，有数据可读，读事件一直触发<br>socket发送缓冲区不满，可以继续写入数据 写事件一直触发<br/>边缘触发(edge-triggered)：<br/>socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件<br/>socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件<br/>边沿触发仅触发一次，水平触发会一直触发。使用EPOLLET标志的应用程序应该使用非阻塞的文件描述符，以避免正在处理多个文件描述符的任务因阻塞读或写而挨饿。多线/进程调用epoll_wait阻塞，边缘触发只会唤醒一个线/进程。水平触发相当于更快的poll。<br>例子如下，边缘触发会导致步骤5阻塞，步骤4只读取了1 kB，即使剩下1 kB但状态没有改变，只有新的写入才会触发：<br/>1. 表示管道读端(rfd)的文件描述符在epoll实例上注册。<br>2. 管道写入器在管道的写端写入2 kB的数据。<br/>3. 调用epoll_wait(2)将返回rfd作为就绪文件描述符。<br/>4. 管道读取器从rfd读取1 kB的数据。<br/>5. 调用epoll_wait(2)。 |

3. 等待事件的产生

   ```c
   #include <sys/epoll.h>
   
   int epoll_wait(int epfd, struct epoll_event *events,
                  int maxevents, int timeout);
   // sigmask指定屏蔽的信号，为NULL时同epoll_wait
   int epoll_pwait(int epfd, struct epoll_event *events,
                   int maxevents, int timeout, const sigset_t *sigmask);
   // struct timespec使时间精度更高
   int epoll_pwait2(int epfd, struct epoll_event *events,
                    int maxevents, const struct timespec *timeout,
                    const sigset_t *sigmask);
   ```

   参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，小于0时将阻塞直到某个事件发生），时间是根据CLOCK_MONOTONIC时钟来测量的。调用成功，返回就绪的数量；调用失败，返回-1并设置errno。

**epoll相比select/poll的优势**：

- select/poll每次调用都要传递所要监控的所有fd给select/poll系统调用（这意味着每次调用都要将fd列表从用户态拷贝到内核态，当fd数目很多时，这会造成低效）。而每次调用epoll_wait时（作用相当于调用select/poll），不需要再传递fd列表给内核，因为已经在epoll_ctl中将需要监控的fd告诉了内核（epoll_ctl不需要每次都拷贝所有的fd，只需要进行增量式操作）。所以，在调用epoll_create之后，内核已经在内核态开始准备数据结构存放要监控的fd了。每次epoll_ctl只是对这个数据结构进行简单的维护。
- select/poll一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的，但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。
- 当我们调用epoll_ctl往里塞入百万个fd时，epoll_wait仍然可以飞快的返回，并有效的将发生事件的fd给我们用户。这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的fd外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的fd，大多一次也只返回很少量的准备就绪fd而已，所以，epoll_wait仅需要从内核态copy少量的fd到用户态而已。那么，这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把fd放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个fd的中断到了，就把它放到准备就绪list链表里。所以，当一个fd（例如socket）上有数据到了，内核在把设备（例如网卡）上的数据copy到内核中后就来把fd（socket）插入到准备就绪list链表里了。

### 2. 例子

虽然epoll作为水平触发接口时的用法与poll(2)具有相同的语义，但边缘触发的用法需要更多的澄清，以避免应用程序事件循环中的停顿。在本例中，listener是一个非阻塞套接字，已在其上调用了listen(2)。函数do_use_fd()使用新的ready文件描述符，直到read(2)或write(2)返回EAGAIN为止。事件驱动的状态机应用程序在接收到EAGAIN之后，应该记录它的当前状态，以便在下一次调用do_use_fd()时，它将继续从之前停止的位置读(2)或写(2)。

```c
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

// 设置监听套接字listen_sock的代码省略了：socket()、bind()、listen()
epollfd = epoll_create1(0);
if (epollfd == -1) {
    perror("epoll_create1");
    exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}

for (;;) {
    nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_wait");
        exit(EXIT_FAILURE);
    }

    for (n = 0; n < nfds; ++n) {
        if (events[n].data.fd == listen_sock) {
            conn_sock = accept(listen_sock,
                                (struct sockaddr *) &addr, &addrlen);
            if (conn_sock == -1) {
                perror("accept");
                exit(EXIT_FAILURE);
            }
            setnonblocking(conn_sock);
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = conn_sock;
            if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                        &ev) == -1) {
                perror("epoll_ctl: conn_sock");
                exit(EXIT_FAILURE);
            }
        } else {
            do_use_fd(events[n].data.fd);
        }
    }
}
```

### 3. 问答

1. 如果在epoll实例上两次注册同一个文件描述符会发生什么?

你可能会得到EEXIST。但是，可以向同一个epoll实例添加一个重复的(dup(2)， dup2(2)， fcntl(2) F_DUPFD)文件描述符。如果重复的文件描述符用不同的事件掩码注册，那么这对于过滤事件是一种有用的技术。

2. 两个epoll实例是否可以等待相同的文件描述符?如果是，事件是否报告给两个epoll文件描述符?

是的，事件会报告给双方。然而，要正确地做到这一点，可能需要仔细的编程。

3. epoll文件描述符本身是否可以poll/epoll/select?

是的。如果epoll文件描述符有事件在等待，那么它将表示为可读。

4. 如果试图将epoll文件描述符放入自己的文件描述符集，会发生什么情况?

epoll_ctl(2)调用失败(EINVAL)。但是，您可以在另一个epoll文件描述符集中添加epoll文件描述符。

5. 我可以通过UNIX域套接字向另一个进程发送epoll文件描述符吗?

是的，但是这样做没有意义，因为接收进程将没有兴趣列表中文件描述符的副本。

6. 关闭文件描述符会导致它从所有epoll兴趣列表中删除吗?

是的，但是要注意以下几点。文件描述符是对打开的文件描述的引用(参见open(2))。

每当通过dup(2)、dup2(2)、fcntl(2) F_DUPFD或fork(2)复制一个文件描述符时，就会创建一个引用相同打开的文件描述符的新文件描述符。打开的文件描述符将继续存在，直到所有引用它的文件描述符都被关闭。

只有在所有引用底层打开的文件描述符的文件描述符都关闭之后，文件描述符才会从兴趣列表中删除。这意味着，即使在作为兴趣列表一部分的文件描述符被关闭之后，如果引用相同底层文件描述符的其他文件描述符仍然打开，则可能报告该文件描述符的事件。为了防止这种情况发生，必须在复制文件描述符之前显式地从兴趣列表中删除(使用epoll_ctl(2) EPOLL_CTL_DEL)。或者，应用程序必须确保所有文件描述符都是关闭的(如果使用dup(2)或fork(2)的库函数在幕后复制文件描述符，那么这可能很困难)。

7. 如果在epoll_wait(2)调用之间发生了多个事件，是合并还是单独报告?

它们将被合并。

8. 对文件描述符的操作是否会影响已经收集但尚未报告的事件?

您可以对现有的文件描述符执行两种操作。在这种情况下，删除是没有意义的。Modify将重新读取可用的I/O。

9. 当使用EPOLLET标志(边缘触发行为)时，我需要持续读/写文件描述符直到EAGAIN吗?

从epoll_wait(2)接收事件应该表明这样的文件描述符已经为请求的I/O操作做好了准备。您必须认为它已经准备好了，直到下一次(非阻塞)读/写产生EAGAIN。何时以及如何使用文件描述符完全由您决定。

对于面向包/令牌的文件(例如，数据报套接字，规范模式下的终端)，检测读/写I/O空间结束的唯一方法是继续读/写，直到EAGAIN。

对于面向流的文件(例如，管道、FIFO、流套接字)，还可以通过检查从目标文件描述符读取/写入的数据量来检测读/写I/O空间耗尽的情况。例如，如果通过请求读取一定量的数据调用read(2)，而read(2)返回的字节数较低，那么可以肯定已经耗尽了文件描述符的读I/O空间。

当使用write(2)写时也是如此。(如果不能保证被监视的文件描述符总是引用面向流的文件，请避免使用后一种技术。)

### 4. 可能的陷阱和避免它们的方法

**饥饿(边沿触发)**

如果有大量的I/O空间，那么通过试图耗尽它，其他文件可能不会得到处理，从而导致饥饿。(这个问题不是epoll特有的。)

解决方案是维护一个就绪列表，并在其关联的数据结构中将文件描述符标记为就绪，从而允许应用程序记住需要处理哪些文件，但仍然在所有就绪文件之间轮询。这还支持忽略已经准备好的文件描述符的后续事件。

**如果使用事件缓存**

如果您使用事件缓存或存储从epoll_wait(2)返回的所有文件描述符，那么请确保提供一种动态标记其闭包的方法(即，由前一个事件处理引起的闭包)。假设您从epoll_wait(2)接收到100个事件，在事件#47中，一个条件导致事件#13关闭。如果删除结构并关闭事件#13的文件描述符，那么事件缓存可能仍然表示有事件在等待该文件描述符，从而造成混乱。

一种解决方案是在处理事件47期间调用epoll_ctl(EPOLL_CTL_DEL)来删除文件描述符13并关闭，然后将其关联的数据结构标记为已删除并将其链接到清理列表。如果在批处理过程中发现文件描述符13的另一个事件，则会发现文件描述符先前已被删除，不会出现混淆。

## 2.4 aio

Linux的aio(7)系列系统调用可以异步处理文件和套接字。然而，有一些限制是你需要注意的:

- aio(7)只支持用O_DIRECT打开的文件或以无缓冲模式打开的文件。这无疑是它最大的局限性。通常情况下，并非所有应用程序都希望以无缓冲模式打开文件。


- 即使在无缓冲模式下，如果文件元数据不可用，aio(7)也可以阻塞。它将等待它可用。


- 有些存储设备对请求有固定数量的插槽。如果所有这些插槽都很忙，Aio(7)提交就会阻塞。


- 总共需要复制104个字节来提交和完成。还需要为I/O执行两个不同的系统调用(分别用于提交和完成)。


上述限制在aio(7)子系统中引入了大量的不确定性和性能开销。

## 2.5 io_uring

io_uring作者文档：https://unixism.net/loti/index.html#

文档中的代码：https://github.com/shuveb/loti    https://github.com/shuveb/loti-examples

liburing（io_uring封装库，提供高级接口）：https://github.com/axboe/liburing

io_uring.pdf：https://kernel.dk/io_uring.pdf

其它：https://lwn.net/Articles/776703/

需要linux内核5.10及以上。

### 2.5.1 接口

#### 1. liburing安装

```shell
$ git clone https://github.com/axboe/liburing.git
$ cd liburing
$ ./configure --help

Usage: configure [options]
Options: [defaults in brackets after descriptions]
  --help                   print this message
  --prefix=PATH            install in PATH [/usr]
  --includedir=PATH        install headers in PATH [/usr/include]
  --libdir=PATH            install runtime libraries in PATH [/usr/lib]
  --libdevdir=PATH         install development libraries in PATH [/usr/lib]
  --mandir=PATH            install man pages in PATH [/usr/man]
  --datadir=PATH           install shared data in PATH [/usr/share]
  --cc=CMD                 use CMD as the C compiler
  --cxx=CMD                use CMD as the C++ compiler
  --nolibc                 build liburing without libc
$ ./configure --prefix=/home/zuoc/work/liburing/liburing-build
$ make && make install
$ echo $?
0

gcc program.c -o ./program -I./liburing/src/include/ -L./liburing/src/ -Wall -O2 -D_GNU_SOURCE -luring 
```

#### 2. 原始系统调用

| 函数                                                         | 说明         |
| ------------------------------------------------------------ | ------------ |
| int io_uring_setup(unsigned entries, struct io_uring_params *p); | 设置环       |
| int io_uring_enter(int fd, unsigned to_submit,  unsigned min_complete, unsigned flags, sigset_t *sig); | 进入系统调用 |
| int io_uring_register(int fd, unsigned int opcode, void *arg,  unsigned int nr_args); | 注册         |

#### 3. 支持的功能

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| *struct* io_uring_probe *io_uring_get_probe_ring(struct io_uring *ring); | 返回io_uring_probe结构体，包涵支持的功能，返回值需要用free释放 |
| struct io_uring_probe *io_uring_get_probe(void);             | 与io_uring_get_probe_ring相同，但负责环的初始化和销毁        |
| int io_uring_opcode_supported(struct io_uring_probe *p, int op); | 判断io_uring_probe结构体是否支持op功能                       |
| int io_uring_register_probe(struct io_uring *ring, struct io_uring_probe *p, unsigned nr); | 与io_uring_get_probe_ring相同，但返回值自己提供，如果不是动态分配。可以不用free |
| void io_uring_free_probe(struct io_uring_probe *probe);      | 释放io_uring_probe结构体                                     |

#### 4. 设置和拆卸

| 函数                                                         | 说明                                             |
| ------------------------------------------------------------ | ------------------------------------------------ |
| int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags); | 初始化                                           |
| int io_uring_queue_init_params(unsigned entries, struct io_uring *ring, struct io_uring_params *p); | 使用io_uring_params 结构体初始化                 |
| int io_uring_queue_mmap(int fd, struct io_uring_params *p, struct io_uring *ring); |                                                  |
| int io_uring_ring_dontfork(struct io_uring *ring);           | 如果不希望进程的子进程继承环映射，则使用此调用。 |
| void io_uring_queue_exit(struct io_uring *ring);             | 释放环                                           |

#### 5. 提交

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring); | 获取提交队列条目                                             |
| void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *data); | 条目设置用户数据                                             |
| void io_uring_sqe_set_data64(struct io_uring_sqe *sqe, __u64 data); | 条目设置用户数据，参数为64位数据                             |
| void io_uring_sqe_set_flags(struct io_uring_sqe *sqe, unsigned flags); | 条目设置标识                                                 |
| int io_uring_submit(struct io_uring *ring);                  | 添加多个条目可一次提交                                       |
| int io_uring_submit_and_wait(struct io_uring *ring, unsigned wait_nr); | 提交并等待wait_nr个条目完成                                  |
| int io_uring_get_events(struct io_uring *ring);              | io_uring_get_events(3)函数运行未完成的工作，并将完成事件刷新到CQE环。如果环已满且已溢出，则可能有需要刷新的事件。或者，如果环是用IORING_SETUP_DEFER_TASKRUN标志设置的，那么这将处理未完成的任务，可能会产生更多的cq。 |
| int io_uring_submit_and_get_events(struct io_uring *ring);   | 向提交队列提交请求并刷新完成                                 |
| int io_uring_sqring_wait(struct io_uring *ring);             | 等待SQ环的空闲空间。仅适用于使用SQPOLL -允许调用者等待SQ环中的空间被释放，当内核端线程消耗了一个或多个条目时，就会发生这种情况。如果SQ环当前未满，则不采取任何操作。注意:如果内核不支持此特性，可能返回-EINVAL。 |
| unsigned io_uring_sq_ready(const struct io_uring *ring);     | 返回SQ环中存在的未消耗(如果SQPOLL)或未提交的条目的数量       |
| unsigned io_uring_sq_space_left(const struct io_uring *ring); | 返回SQ环中还剩下多少空间。                                   |

#### 6. 完成

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr); | 等待至少一个完成，否则阻塞                                   |
| int io_uring_wait_cqe_nr(struct io_uring *ring, struct io_uring_cqe **cqe_ptr, unsigned wait_nr); | 等待wait_nr个完成                                            |
| int io_uring_wait_cqes(struct io_uring *ring, struct io_uring_cqe **cqe_ptr, unsigned wait_nr, struct __kernel_timespec *ts, sigset_t *sigmask); | 与io_uring_wait_cqe()类似，只不过它也接受超时值。注意，SQE在内部用于处理超时。使用此函数的应用程序绝对不能将sqe->user_data设置为LIBURING_UDATA_TIMEOUT。<br/>如果指定了ts，应用程序在调用这个函数之前不需要调用io_uring_submit()，因为它将在内部完成。由此还可以推断，对于那些将SQ和CQ处理拆分到两个线程之间的应用程序，并期望在没有同步的情况下工作的应用程序，该函数是不安全的，因为该函数同时操作SQ和CQ端。 |
| int io_uring_wait_cqe_timeout(struct io_uring *ring, struct io_uring_cqe **cqe_ptr, struct __kernel_timespec *ts); | 与io_uring_wait_cqes()相同的是，它不接受sigmask参数，并且总是将wait_nr设置为1。 |
| int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr); | 等待至少一个完成，不阻塞                                     |
| unsigned io_uring_peek_batch_cqe(struct io_uring *ring, struct io_uring_cqe **cqes, unsigned count); | 如果I/O补全可用，则填充到count为止的数组，并返回已填充的补全计数。不等待完成。它们必须已经可用，才能被这个函数返回。 |
| void *io_uring_cqe_get_data(const struct io_uring_cqe *cqe); | 获取用户数据                                                 |
| __u64 io_uring_cqe_get_data64(const struct io_uring_cqe *cqe); | 获取用户数据，返回值为64位                                   |
| void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe); | 将条目取出。必须在io_uring_peek_cqe()或io_uring_wait_cqe()之后以及在cqe已被应用程序处理之后调用。 |
| void io_uring_cq_advance(struct io_uring *ring, unsigned nr); | io_uring_cq_advance(3)函数将属于ring参数的nr个IO补全标记为已消耗。函数io_uring_cqe_seen(3)调用函数io_uring_cq_advance(3)。 |
| unsigned io_uring_cq_ready(struct io_uring *ring);           | 返回CQ环中未使用的就绪项的个数                               |
| bool io_uring_cq_has_overflow(const struct io_uring *ring);  | 如果有溢出项等待刷新到CQ环上，则返回true                     |
| struct cmsghdr *io_uring_recvmsg_cmsg_firsthdr(struct io_uring_recvmsg_out *o, struct msghdr *msgh); | 从multishot recvmsg获取数据                                  |
| struct cmsghdr *io_uring_recvmsg_cmsg_nexthdr(struct io_uring_recvmsg_out *o, struct msghdr *msgh, struct cmsghdr *cmsg); | 从multishot recvmsg获取数据                                  |
| void *io_uring_recvmsg_name(struct io_uring_recvmsg_out *o); | 从multishot recvmsg获取数据                                  |
| void *io_uring_recvmsg_payload(struct io_uring_recvmsg_out *o, struct msghdr *msgh); | 从multishot recvmsg获取数据                                  |
| unsigned int io_uring_recvmsg_payload_length(struct io_uring_recvmsg_out *o, int buf_len, struct msghdr *msgh); | 从multishot recvmsg获取数据                                  |
| struct io_uring_recvmsg_out *io_uring_recvmsg_validate(void *buf, int buf_len, struct msghdr *msgh); | 从multishot recvmsg获取数据                                  |

#### 7. 提交助手，对应IORING_OP_XXX

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void io_uring_prep_nop(*struct* io_uring_sqe **sqe*);        | 该函数使用IORING_OP_NOP操作设置sqe指向的提交队列条目，这是一个no-op。这种操作的存在是为了测试目的，用于测试io_uring接口的速度和效率。 |
| void io_uring_prep_read(*struct* io_uring_sqe *sqe, int *fd*, void **buf*, unsigned *nbytes*, off_t *offset*); | 从fd的offset偏移处读nbytes字节到buf                          |
| void io_uring_prep_write(*struct* io_uring_sqe *sqe, int *fd*, *const* void **buf*, unsigned *nbytes*, off_t *offset*); | 向fd的offset偏移处写nbytes字节buf数据                        |
| void io_uring_prep_readv(*struct* io_uring_sqe *sqe, int *fd*, *const* *struct* iovec **iovecs*, unsigned *nr_vecs*, off_t *offset*); | 散布读                                                       |
| void io_uring_prep_readv2(struct io_uring_sqe *sqe, int fd, const struct iovec *iovecs, unsigned nr_vecs, __u64 offset, int flags); | 散布读，添加了flags： <br/>RWF_HIPRI：高优先级请求，如果可能的话轮询<br/>RWF_DSYNC：per-IO O_DSYNC<br/>RWF_SYNC：per-IO O_SYNC<br/>RWF_NOWAIT：per-IO, 操作可能阻塞则返回-EAGAIN<br/>RWF_APPEND：per-IO O_APPEND |
| void io_uring_prep_read_fixed(*struct* io_uring_sqe *sqe, int *fd*, void **buf*, unsigned *nbytes*, off_t *offset*, int *buf_index*); | 与io_uring_prep_read()非常类似，该函数使用读操作设置sqe指向的提交队列条目。主要的区别是，该函数被设计为使用通过io_uring_register_buffers()注册的固定预分配缓冲区。<br>buf和nbytes参数必须在前面注册的缓冲区中由buf_index指定的区域内。缓冲区不需要与已注册缓冲区的开始对齐。 |
| void io_uring_prep_writev(*struct* io_uring_sqe *sqe, int *fd*, *const* *struct* iovec **iovecs*, unsigned *nr_vecs*, off_t *offset*); | 聚集写                                                       |
| void io_uring_prep_writev2(struct io_uring_sqe *sqe, int fd, const struct iovec *iovecs, unsigned nr_vecs, __u64 offset, int flags); | 聚集写，添加了flags                                          |
| void io_uring_prep_write_fixed(*struct* io_uring_sqe *sqe, int *fd*, *const* void **buf*, unsigned *nbytes*, off_t *offset*, int *buf_index*); | 同io_uring_prep_read_fixed()                                 |
| void io_uring_prep_fsync(*struct* io_uring_sqe **sqe*, int *fd*, unsigned *fsync_flags*); | 该函数使用类似fsync(2)的操作设置sqe指向的提交队列条目。这将导致磁盘缓存中文件数据和元数据的任何“脏”缓冲区被同步到磁盘。注意：将此操作排队并不能保证在此操作之前排队的任何写操作将使写入文件的数据同步到磁盘。<br>fsync_flags: 它可以是0或' IORING_FSYNC_DATASYNC '，这使它的行为类似于fdatasync(2)。 |
| void io_uring_prep_close(*struct* io_uring_sqe **sqe*, int *fd*); | 关闭描述符                                                   |
| void io_uring_prep_close_direct(struct io_uring_sqe *sqe, unsigned file_index); | 对于直接描述符关闭请求，偏移量由file_index参数而不是fd参数指定。这与取消注册直接描述符是相同的，提供这种方法是为了方便。 |
| void io_uring_prep_openat(*struct* io_uring_sqe *sqe, int *dfd*, *const* char **path*, int *flags*, mode_t *mode*); | 类似于openat(2)                                              |
| void io_uring_prep_openat_direct(struct io_uring_sqe *sqe, int flags, mode_t mode, unsigned file_index); | io_uring_prep_openat的direct版本                             |
| void io_uring_prep_openat2(*struct* io_uring_sqe *sqe, int *dfd*, *const* char *path, *struct* open_how **how*); | 类似于openat2(2)                                             |
| void io_uring_prep_openat2_direct(struct io_uring_sqe *sqe, int dfd, const char *path, struct open_how *how, unsigned file_index); | io_uring_prep_openat2的direct版本                            |
| void io_uring_prep_fallocate(*struct* io_uring_sqe **sqe*, int *fd*, int *mode*, off_t *offset*, off_t *len*); | 该函数使用类似fallocate(2)的操作设置sqe指向的提交队列条目。fallocate(2)系统调用用于为文件描述符fd表示的文件分配、释放、折叠、置零或增加文件空间。 |
| void io_uring_prep_statx(*struct* io_uring_sqe *sqe, int *dfd*, *const* char *path, int *flags*, unsigned *mask*, *struct* statx **statxbuf*); | 该函数使用类似statx(2)的操作设置sqe指向的提交队列条目。statx(2)系统调用获取由路径指向的文件的元信息，该信息被填充到由statxbuf指向的statx结构中。 |
| void io_uring_prep_fadvise(*struct* io_uring_sqe **sqe*, int *fd*, off_t *offset*, off_t *len*, int *advice*); | 这个函数使用类似posix_fadvise(2)的操作设置sqe指向的提交队列条目。posix_fadvise(2)系统调用允许应用程序通知操作系统它计划如何访问文件描述符fd表示的文件中的数据，顺序、随机或以其他方式。这是为了提高应用程序的性能。 |
| void io_uring_prep_madvise(*struct* io_uring_sqe *sqe, void *addr, off_t *length*, int *advice*); | 该函数使用类似madvise(2)的操作设置sqe指向的提交队列条目。madvise(2)系统调用允许应用程序将由addr和length指向的内存通知给操作系统。这些建议可能是关于应用程序计划如何访问上述内存范围(顺序、随机或其他方式)，或者在进程fork子进程时操作系统是否应该共享它等等。这是为了提高应用程序的性能。 |
| void io_uring_prep_splice(*struct* io_uring_sqe **sqe*, int *fd_in*, loff_t off_in, int *fd_out*, loff_t *off_out*, unsigned int *nbytes*, unsigned int *splice_flags*); | 该函数使用类似于splice(2)的操作设置sqe指向的提交队列条目。splice(2)系统调用在两个文件描述符(fd_in和fd_out)之间复制数据，而不在内核地址空间和用户地址空间之间复制数据。但是，必须有一个文件描述符表示管道。 |
| void io_uring_prep_recvmsg(*struct* io_uring_sqe *sqe, int *fd*, *struct* msghdr **msg*, unsigned *flags*); | 该函数使用类似recvmsg(2)的操作设置sqe指向的提交队列条目。recvmsg(2)系统调用用于从套接字读取数据。它使用msghdr结构来减少参数的数量。此调用可用于面向连接(如TCP)和无连接(如UDP)套接字。使用*struct* msghdr类似于readv。 |
| void io_uring_prep_recvmsg_multishot(struct io_uring_sqe *sqe, int fd, struct msghdr *msg, unsigned flags); |                                                              |
| void io_uring_prep_sendmsg(*struct* io_uring_sqe *sqe, int *fd*, const struct msghdr *msg, unsigned flags); | 类似于sendmsg(2)                                             |
| void io_uring_prep_sendmsg_zc(struct io_uring_sqe *sqe, int fd, const struct msghdr *msg, unsigned flags); | io_uring_prep_sendmsg零拷贝版本                              |
| void io_uring_prep_send_set_addr(struct io_uring_sqe *sqe, const struct sockaddr *dest_addr, __u16 addr_len); | 设置发送的目标地址                                           |
| void io_uring_prep_recv(*struct* io_uring_sqe *sqe, int *sockfd*, void *buf, size_t *len*, int *flags*); | 类似于recv(2)                                                |
| void io_uring_prep_send(*struct* io_uring_sqe *sqe, int *sockfd*, *const* void **buf*, size_t *len*, int *flags*); | 类似于send(2)                                                |
| void io_uring_prep_send_zc(struct io_uring_sqe *sqe, int sockfd, const void *buf, size_t len, int flags, unsigned zc_flags); | 准备一个零拷贝发送请求。提交队列条目sqe被设置为使用文件描述符sockfd开始从大小为len bytes的buf发送数据，并带有send修饰符flags flags和zerocopy修饰符flags zc_flags。 |
| void io_uring_prep_send_zc_fixed(struct io_uring_sqe *sqe, int sockfd, const void *buf, size_t len, int flags, unsigned zc_flags, unsigned buf_index); | 零拷贝发送请求的fixed版本。                                  |
| void io_uring_prep_accept(*struct* io_uring_sqe *sqe, int *fd*, *struct* sockaddr *addr, socklen_t *addrlen, int *flags*); | 该函数使用类似accept4(2)的操作设置sqe指向的提交队列条目。accept4(2)系统调用用于面向连接的套接字类型(SOCK_STREAM, SOCK_SEQPACKET)。它为正在监听的套接字fd提取挂起连接队列上的第一个连接请求。当flags参数设置为0时，accept4(2)与accept(2)完全等价。 |
| void io_uring_prep_accept_direct(struct io_uring_sqe *sqe, int fd, struct sockaddr *addr, socklen_t *addrlen, int flags, unsigned int file_index); | 直接接受到固定文件表中                                       |
| void io_uring_prep_multishot_accept(struct io_uring_sqe *sqe, int fd, struct sockaddr *addr, socklen_t *addrlen, int flags); | 多镜头版本accept和accept_direct允许应用程序发出单个accept请求，当连接请求传入时，该请求将重复触发CQE。与其他多射类型请求一样，应用程序应该查看CQE标志，并查看是否在完成时设置了IORING_CQE_F_MORE，以指示接受的请求是否将生成更多的CQE。注意，对于多镜头变量，设置addr和addrlen可能没有太大意义，因为每个接受的连接都将使用相同的值。这意味着，在应用程序有时间处理过去的连接之前，写入addr的数据可能会被新连接覆盖。如果应用程序知道在处理前一个连接之前不能传入新的连接，则可以按预期使用它。自5.19起可用多重射击版本。 |
| void io_uring_prep_multishot_accept_direct(struct io_uring_sqe *sqe, int fd, struct sockaddr *addr, socklen_t *addrlen, int flags); | io_uring_prep_multishot_accept的直接描述符版本               |
| void io_uring_prep_connect(*struct* io_uring_sqe *sqe, int *fd*, *struct* sockaddr **addr*, socklen_t *addrlen*); | 类似于connect(2)                                             |
| void io_uring_prep_epoll_ctl(*struct* io_uring_sqe *sqe, int *epfd*, int *fd*, int *op*, *struct* epoll_event **ev*); | 该函数使用类似epoll_ctl(2)的操作设置sqe指向的提交队列条目。epoll_ctl(2)系统调用用于在epfd引用的epoll(7)实例的兴趣列表中添加或删除修改项。由op指定的添加、删除或修改操作应用于文件描述符fd。<br>op: EPOLL_CTL_ADD(添加)、EPOLL_CTL_DEL(删除)、EPOLL_CTL_MOD(修改) |
| void io_uring_prep_poll_add(*struct* io_uring_sqe **sqe*, int *fd*, short *poll_mask*); | 该函数使用类似poll(2)的操作设置sqe指向的提交队列条目，将文件描述符添加到poll的兴趣列表中，并监听poll_mask中指定的事件。与没有EPOLLONESHOT的poll或epoll不同，该接口总是在一次性模式下工作。也就是说，一旦poll操作完成，就必须重新提交。 |
| void io_uring_prep_poll_remove(*struct* io_uring_sqe *sqe, void *user_data); | 通过poll(2)从监视请求中删除。user_data指针指向用户数据，与此用户数据关联的请求将从进一步监视中删除。 |
| void io_uring_prep_poll_update(struct io_uring_sqe *sqe, \__u64 old_user_data, __u64 new_user_data, unsigned poll_mask, unsigned flags); | poll更新                                                     |
| void io_uring_prep_poll_multishot(struct io_uring_sqe *sqe, int fd, unsigned poll_mask); | 同io_uring_prep_poll_add(3)，默认行为是单次轮询请求。当指定的事件被触发时，将发布一个完成CQE，并且轮询请求将不再生成任何事件。Io_uring_prep_multishot(3)在事件方面的行为是相同的，但它在通知之间持续存在，并会重复发布同一注册的通知。从多次轮询请求中发布的CQE将在CQE标志成员中设置IORING_CQE_F_MORE，这表明应用程序应该期望从该请求中完成更多的操作。如果多次轮询请求被终止或遇到错误，则不会在CQE中设置此标志。如果发生这种情况，应用程序不应该期望从原始请求获得更多的cq，如果它仍然希望获得关于该文件描述符的通知，则必须重新发出一个新的cq。 |
| void io_uring_prep_sync_file_range(struct io_uring_sqe *sqe, int fd, unsigned len,__u64 offset, int flags); | 类似于sync_file_range(2)。Sync_file_range()允许在将文件描述符fd引用的打开文件与磁盘同步时进行精细控制。flags位掩码参数可以包括以下任何值:<br>SYNC_FILE_RANGE_WAIT_BEFORE：在执行任何写操作之前，等待指定范围内已经提交给设备驱动程序的所有页面被写出。<br/>SYNC_FILE_RANGE_WRITE：启动写出指定范围内当前未提交写出的所有脏页。注意，如果您试图写入超过请求队列大小的数据，即使这样也可能阻塞。<br/>SYNC_FILE_RANGE_WAIT_AFTER：在执行任何写操作后，等待写出范围内的所有页。<br/>允许将标志指定为0，作为无操作。 |
| void io_uring_prep_timeout(struct io_uring_sqe *sqe, struct __kernel_timespec *ts, unsigned count, unsigned flags); | 准备一个超时请求。如果指定的超时已经发生，或者等待的指定数量的事件已经发布到CQ环，则超时完成事件将被触发。 |
| void io_uring_prep_timeout_update(struct io_uring_sqe *sqe, struct \_\_kernel_timespec *ts, __u64 user_data, unsigned flags); | 准备一个更新现有超时的请求，user_data指定超时。              |
| void io_uring_prep_timeout_remove(struct io_uring_sqe *sqe, __u64 user_data, unsigned flags); | 准备一个删除现有超时的请求，user_data指定超时。              |
| void io_uring_prep_cancel64(struct io_uring_sqe *sqe, __u64 user_data, int flags); | 与io_uring_prep_cancel(3)相同，不同之处是它接受64位整数而不是指针类型。 |
| void io_uring_prep_cancel(struct io_uring_sqe *sqe, void *user_data, int flags); | 取消由user_data标识的现有请求。默认情况下，第一个符合条件的请求将被取消。可以通过传入的flags进行修改:<br/>IORING_ASYNC_CANCEL_ALL：取消所有符合给定条件的请求，而不是只取消找到的第一个请求。5.19后可用。<br/>IORING_ASYNC_CANCEL_FD：根据原始请求中使用的文件描述符而不是user_data进行匹配。这就是io_uring_prep_cancel_fd(3)所设置的。5.19后可用。<br/>IORING_ASYNC_CANCEL_ANY：匹配环中的任何请求，不管user_data或文件描述符是什么。可用于取消环中任何挂起的请求。5.19后可用。 |
| void io_uring_prep_cancel_fd(struct io_uring_sqe *sqe, int fd, unsigned int flags); | 取消由fd标识的现有请求。                                     |
| void io_uring_prep_link_timeout(struct io_uring_sqe *sqe, struct __kernel_timespec *ts, unsigned flags); | IORING_OP_LINK_TIMEOUT                                       |
| void io_uring_prep_files_update(struct io_uring_sqe *sqe, int *fds, unsigned nr_fds, int offset); | io_uring_prep_files_update(3)函数的作用是:准备一个更新以前注册的文件描述符的请求。将提交队列条目sqe设置为使用fds指向的文件描述符数组和长度为nr_fds的文件描述符数组来更新从offset开始的先前注册的文件数量。<br/>一旦用新文件更新了以前注册的文件，就会更新现有的条目，然后从表中删除。这个操作相当于首先取消该条目的注册，然后插入一个新的条目，只是绑定到一个组合操作中。<br/>如果offset被指定为IORING_FILE_INDEX_ALLOC, io_uring将分配空闲的直接描述符而不是让应用程序传递，并将分配的直接描述符存储到fds数组中，cqe->res将返回分配的直接描述符的数量。 |
| void io_uring_prep_provide_buffers(struct io_uring_sqe *sqe, void *addr, int len, int nr,int bgid, int bid); | 向内核提供缓冲区准备了一个请求。提交队列条目sqe被设置为使用len数量的缓冲区，从addr开始，由缓冲区组ID bgid标识，从bid开始依次编号。 |
| void io_uring_prep_remove_buffers(struct io_uring_sqe *sqe, int nr, int bgid); | 准备删除先前提供的缓冲区的请求。设置提交队列条目sqe以从bgid指示的缓冲组ID中删除nr个缓冲区。 |
| void io_uring_prep_tee(struct io_uring_sqe *sqe, int fd_in, int fd_out, unsigned int nbytes, unsigned int splice_flags); | 准备异步tee(2)请求，tee将标准输入同时复制到标准输出和命令行中的文件中。将提交队列条目sqe设置为使用文件描述符fd_in作为输入，使用文件描述符fd_out作为输出，复制nbytes字节的数据。Splice_flags是操作的修饰符标志。参见tee(2)了解通用拼接标志。<br/>如果fd_out描述符，那么可以在SQE中设置IOSQE_FIXED_FILE来指示这一点。对于输入文件，可以设置io_uring特定的SPLICE_F_FD_IN_FIXED，并将fd_in作为注册的文件描述符偏移量。 |
| void io_uring_prep_shutdown(struct io_uring_sqe *sqe, int fd, int how); | 类似于shutdown(2)，关闭套接字的读端、写段或读写端            |
| void io_uring_prep_renameat(struct io_uring_sqe *sqe, int olddfd, const char *oldpath, int newdfd, const char *newpath, unsigned int flags); | 异步renameat2(2)请求。                                       |
| void io_uring_prep_rename(struct io_uring_sqe *sqe, const char *oldpath, const char *newpath); | 异步rename(2)请求。                                          |
| void io_uring_prep_unlinkat(struct io_uring_sqe *sqe, int dfd, const char *path, int flags); | 异步unlinkat(2)请求                                          |
| void io_uring_prep_unlink(struct io_uring_sqe *sqe, const char *path, int flags); | 异步unlink(2)请求                                            |
| void io_uring_prep_mkdirat(struct io_uring_sqe *sqe, int dfd, const char *path, mode_t mode); | 异步mkdiratat(2)请求                                         |
| void io_uring_prep_mkdir(struct io_uring_sqe *sqe, const char *path, mode_t mode); | 异步mkdirat(2)请求                                           |
| void io_uring_prep_symlinkat(struct io_uring_sqe *sqe, const char *target, int newdirfd, const char *linkpath); | 异步symlinkat(2)请求                                         |
| void io_uring_prep_symlink(struct io_uring_sqe *sqe, const char *target, const char *linkpath); | 异步symlink(2)请求                                           |
| void io_uring_prep_linkat(struct io_uring_sqe *sqe, int olddfd, const char *oldpath, int newdfd, const char *newpath, int flags); | 异步linkat(2)请求                                            |
| void io_uring_prep_link(struct io_uring_sqe *sqe, const char *oldpath, const char *newpath, int flags); | 异步link(2)请求                                              |
| void io_uring_prep_msg_ring(struct io_uring_sqe *sqe, int fd, unsigned int len, __u64 data, unsigned int flags); | 准备向io_uring文件描述符发送CQE。提交队列条目sqe被设置为使用文件描述符fd(必须标识io_uring上下文)在该环上发布一个CQE，其中目标CQE res字段将包含len的内容和数据的user_data，并使用flags设置的请求修饰符flags。目前没有有效的flag修饰符，该字段必须包含0。<br/>目标ring可以是用户可以访问的任何ring，甚至是ring本身。此请求可用于将简单的消息传递到另一个ring，允许通过len和data字段传输32+64位的数据。用例可以是简单地唤醒在目标环上等待的人，也可以用于在两个环之间传递消息。 |
| void io_uring_prep_fsetxattr(struct io_uring_sqe *sqe, int fd, const char *name, const char	*value, int	flags, unsigned int len); | IORING_OP_FSETXATTR                                          |
| void io_uring_prep_setxattr(struct io_uring_sqe *sqe, const char *name, const char *value, const char *path, int flags, unsigned int len); | IORING_OP_SETXATTR                                           |
| void io_uring_prep_fgetxattr(struct io_uring_sqe *sqe, int fd, const char *name, char *value, unsigned int len); | IORING_OP_FGETXATTR                                          |
| void io_uring_prep_getxattr(struct io_uring_sqe *sqe, const char *name, char *value, const char *path, unsigned int len); | IORING_OP_GETXATTR                                           |
| void io_uring_prep_socket(struct io_uring_sqe *sqe, int domain, int type, int protocol, unsigned int flags); | 准备一个套接字创建请求。提交队列入口sqe被设置为使用通信domain由域定义，使用由type定义的通信类型和由protocol设置的协议。flags参数目前未使用。 |
| void io_uring_prep_socket_direct(struct io_uring_sqe *sqe, int domain, int type, int protocol, unsigned file_index, unsigned int flags); | 工作原理与io_uring_prep_socket(3)类似，只是它将套接字映射到一个直接描述符，而不是返回一个普通的文件描述符。file_index参数应该设置为应该用于该套接字的槽位。 |
| void io_uring_prep_socket_direct_alloc(struct io_uring_sqe *sqe, int domain, int type, int protocol, unsigned int flags); | 工作原理与io_uring_prep_socket(3)类似，只是它分配了一个新的直接描述符而不是传递一个空闲的插槽。它等价于使用io_uring_prep_socket_direct(3)和IORING_FILE_INDEX_ALLOC作为file_index。完成后，CQE的res字段将返回分配给套接字的直接槽。 |

#### 8. 其它方法

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| int io_uring_register_buffers(struct io_uring *ring, const struct iovec *iovecs, unsigned nr_iovecs); | 注册缓冲区用于固定缓冲区操作，如io_uring_prep_write_fixed()和io_uring_prep_read_fixed()。 |
| int io_uring_unregister_buffers(struct io_uring *ring);      | 取消注册缓冲区。                                             |
| int io_uring_register_files(struct io_uring *ring, const int *files, unsigned nr_files); | 注册后续操作中使用的文件描述符，比如那些涉及提交队列轮询(IORING_SETUP_SQPOLL)的操作。 |
| int io_uring_unregister_files(struct io_uring *ring);        | 取消注册文件描述符。                                         |
| int io_uring_register_files_update(struct io_uring *ring, unsigned off, int *files, unsigned nr_files); | 更新注册文件描述符。                                         |
| int io_uring_register_eventfd(struct io_uring *ring, int fd); | 通过向io_uring注册一个eventfd(2)文件描述符，就可以在io_uring实例上获得完成事件的通知，而这个函数可以让你做到这一点。 |
| int io_uring_register_eventfd_async(struct io_uring *ring, int fd); | 与io_uring_register_eventfd()相同，只是通知只发布在异步方式完成的事件上。这意味着在提交时完成内联的事件不会触发通知事件。 |
| int io_uring_unregister_eventfd(struct io_uring *ring);      | 取消注册事件描述符。                                         |
| int io_uring_register_personality(struct io_uring *ring);    | 该操作用io_uring注册正在运行的应用程序的凭据，并返回与这些凭据相关联的ID。希望在不同的用户/进程之间共享一个环的应用程序可以在SQE个性字段中传入这个凭据ID。如果设置，该特定sqe将与这些凭证一起颁发。 |
| int io_uring_unregister_personality(struct io_uring *ring, int id); | 取消注册凭据。                                               |
| void io_uring_buf_ring_add(struct io_uring_buf_ring *br, void *addr, unsigned int len, unsigned short bid, int mask, int buf_offset); | 将一个新的缓冲区添加到共享缓冲区环br中。缓冲区地址由addr表示，长度为len字节。bid是缓冲区ID，它将在CQE中返回。Mask是环的大小掩码，可从io_uring_buf_ring_mask(3)获得。buf_offset是从当前尾部插入的偏移量。如果在io_uring_buf_ring_advance(3)或io_uring_buf_ring_cq_advance(3)提交环尾之前只提供了一个缓冲区，那么buf_offset应该是0。如果在提交之前在循环中提供了缓冲区，buf_offset必须为每添加一个缓冲区加1。 |
| void io_uring_buf_ring_advance(struct io_uring_buf_ring *br, int count); | 让'count'新缓冲区对内核可见。在io_uring_buf_ring_add()被调用'count'次后调用，以填充新的缓冲区。 |
| void io_uring_buf_ring_cq_advance(struct io_uring *ring, struct io_uring_buf_ring *br, int count); | 提交计数以前添加到共享缓冲区环br的缓冲区，使它们对内核可见，因此可以使用。这将缓冲区的所有权传递给环。同时，提高了环的CQ环的计数量。这有效地将io_uring_buf_ring_advance(3)调用和io_uring_cq_avance(3)绑定到一个操作中。由于更新任何一个环索引都需要存储内存屏障，因此同时更新两个环索引效率更高。 |
| int io_uring_register_buf_ring(struct io_uring *ring, struct io_uring_buf_reg *reg, unsigned int flags); | 为所提供的缓冲区注册缓冲环                                   |
| int io_uring_unregister_buf_ring(struct io_uring *ring, int bgid); | 注销io_uring_register_buf_ring                               |
| void io_uring_buf_ring_init(struct io_uring_buf_ring *br);   | 初始化br以便它可以被使用。它可以在io_uring_register_buf_ring(3)之后调用，但必须在以任何其他方式使用缓冲环之前调用。 |
| int io_uring_buf_ring_mask(__u32 ring_entries);              | 返回大小为'ring_entries'的缓冲区环的适当掩码                 |
| int io_uring_register_iowq_aff(struct io_uring *ring, size_t cpusz, const cpu_set_t *mask); | 注册async worker CPU关联                                     |
| int io_uring_unregister_iowq_aff(struct io_uring *ring);     | 注销io_uring_register_iowq_aff                               |
| int io_uring_register_iowq_max_workers(struct io_uring *ring, unsigned int *values); | 修改允许的最大异步工作线程                                   |
| int io_uring_register_ring_fd(struct io_uring *ring);        | 注册ring文件描述符                                           |
| int io_uring_unregister_ring_fd(struct io_uring *ring);      | 注销io_uring_register_ring_fd                                |
| int io_uring_register_sync_cancel(struct io_uring *ring, struct io_uring_sync_cancel_reg *reg); | 发出同步取消请求                                             |

### 2.5.2 内核头文件

```c
typedef __signed__ char __s8;
typedef unsigned char __u8;

typedef __signed__ short __s16;
typedef unsigned short __u16;

typedef __signed__ int __s32;
typedef unsigned int __u32;

typedef __signed__ long __s64;
typedef unsigned long __u64;
```

以下头文件拷贝自Linux内核v5.19.16

#### 1. /include/uapi/linux/io_uring.h

```c
// io_uring接口头文件
#ifndef LINUX_IO_URING_H
#define LINUX_IO_URING_H

#include <linux/fs.h>
#include <linux/types.h>

// IO submission data structure (Submission Queue Entry)
// IO提交数据结构(提交队列条目)
struct io_uring_sqe {
	__u8	opcode;		/* type of operation for this sqe 此sqe的操作类型 */
	__u8	flags;		/* IOSQE_ flags */
	__u16	ioprio;		/* ioprio for the request */
	__s32	fd;		/* file descriptor to do IO on 进行IO的文件描述符 */
	union {
		__u64	off;	/* offset into file */
		__u64	addr2;
		struct {
			__u32	cmd_op;
			__u32	__pad1;
		};
	};
	union {
		__u64	addr;	/* pointer to buffer or iovecs 指向缓冲区或iovecs的指针 */
		__u64	splice_off_in;
	};
	__u32	len;		/* buffer size or number of iovecs 缓冲区大小或iovecs数量 */
	union {
		__kernel_rwf_t	rw_flags;
		__u32		fsync_flags;
		__u16		poll_events;	/* compatibility 兼容性 */
		__u32		poll32_events;	/* word-reversed for BE */
		__u32		sync_range_flags;
		__u32		msg_flags;
		__u32		timeout_flags;
		__u32		accept_flags;
		__u32		cancel_flags;
		__u32		open_flags;
		__u32		statx_flags;
		__u32		fadvise_advice;
		__u32		splice_flags;
		__u32		rename_flags;
		__u32		unlink_flags;
		__u32		hardlink_flags;
		__u32		xattr_flags;
	};
	__u64	user_data;	/* data to be passed back at completion time 在完成时返回的数据 */
	/* pack this to avoid bogus arm OABI complaints 打包这个以避免虚假的arm OABI投诉 */
	union {
		/* index into fixed buffers, if used 固定缓冲区的索引，如果使用 */
		__u16	buf_index;
		/* for grouped buffer selection 对于分组的缓冲区选择 */
		__u16	buf_group;
	} __attribute__((packed));
	/* personality to use, if used 使用的个性，如果使用 */
	__u16	personality;
	union {
		__s32	splice_fd_in;
		__u32	file_index;
	};
	union {
		struct {
			__u64	addr3;
			__u64	__pad2[1];
		};
		// 如果用IORING_SETUP_SQE128初始化环，则该字段用于80字节的任意命令数据
		__u8	cmd[0];
	};
};

/*
 * If sqe->file_index is set to this for opcodes that instantiate a new
 * direct descriptor (like openat/openat2/accept), then io_uring will allocate
 * an available direct descriptor instead of having the application pass one
 * in. The picked direct descriptor will be returned in cqe->res, or -ENFILE
 * if the space is full.
 */
// 如果sqe->file_index对实例化一个新的直接描述符的操作码设置为这个(如openat/openat2/accept)，那么io_uring将分配一个可用的直接描述符，而不是让应用程序传入一个。所选的直接描述符将在cqe->res中返回，如果空间已满则返回-ENFILE。
#define IORING_FILE_INDEX_ALLOC		(~0U)

enum {
	IOSQE_FIXED_FILE_BIT,
	IOSQE_IO_DRAIN_BIT,
	IOSQE_IO_LINK_BIT,
	IOSQE_IO_HARDLINK_BIT,
	IOSQE_ASYNC_BIT,
	IOSQE_BUFFER_SELECT_BIT,
	IOSQE_CQE_SKIP_SUCCESS_BIT,
};

/*
 * sqe->flags
 */
/* use fixed fileset 使用固定文件集 */
#define IOSQE_FIXED_FILE	(1U << IOSQE_FIXED_FILE_BIT)
/* issue after inflight IO 飞行IO后出现问题 */
#define IOSQE_IO_DRAIN		(1U << IOSQE_IO_DRAIN_BIT)
/* links next sqe 链接下个sqe */
#define IOSQE_IO_LINK		(1U << IOSQE_IO_LINK_BIT)
/* like LINK, but stronger 像LINK，但更强 */
#define IOSQE_IO_HARDLINK	(1U << IOSQE_IO_HARDLINK_BIT)
/* always go async 总是异步 */
#define IOSQE_ASYNC		(1U << IOSQE_ASYNC_BIT)
/* select buffer from sqe->buf_group 从sqe->buf_group选择缓冲区 */
#define IOSQE_BUFFER_SELECT	(1U << IOSQE_BUFFER_SELECT_BIT)
/* don't post CQE if request succeeded 如果请求成功，不要发布CQE */
#define IOSQE_CQE_SKIP_SUCCESS	(1U << IOSQE_CQE_SKIP_SUCCESS_BIT)

/*
 * io_uring_setup() flags
 */
#define IORING_SETUP_IOPOLL	(1U << 0)	/* io_context is polled io_context已轮询 */
#define IORING_SETUP_SQPOLL	(1U << 1)	/* SQ poll thread SQ轮询线程 */
#define IORING_SETUP_SQ_AFF	(1U << 2)	/* sq_thread_cpu is valid sq_thread_cpu是有效的 */
#define IORING_SETUP_CQSIZE	(1U << 3)	/* app defines CQ size app定义CQ大小 */
#define IORING_SETUP_CLAMP	(1U << 4)	/* clamp SQ/CQ ring sizes 设定SQ/CQ环缓冲大小 */
#define IORING_SETUP_ATTACH_WQ	(1U << 5)	/* attach to existing wq 附加到已有的wq */
#define IORING_SETUP_R_DISABLED	(1U << 6)	/* start with ring disabled 开始时禁用环 */
#define IORING_SETUP_SUBMIT_ALL	(1U << 7)	/* continue submit on error 出现错误时继续提交 */
/*
 * Cooperative task running. When requests complete, they often require
 * forcing the submitter to transition to the kernel to complete. If this
 * flag is set, work will be done when the task transitions anyway, rather
 * than force an inter-processor interrupt reschedule. This avoids interrupting
 * a task running in userspace, and saves an IPI.
 */
// 合作任务运行。当请求完成时，它们通常需要强制提交者转换到内核来完成。如果设置了这个标志，工作将在任务转换时完成，而不是强制处理器间中断重新调度。这避免了中断用户空间中运行的任务，并节省了一个IPI。
#define IORING_SETUP_COOP_TASKRUN	(1U << 8)
/*
 * If COOP_TASKRUN is set, get notified if task work is available for
 * running and a kernel transition would be needed to run it. This sets
 * IORING_SQ_TASKRUN in the sq ring flags. Not valid with COOP_TASKRUN.
 */
// 如果设置了COOP_TASKRUN，那么如果任务工作可以运行并且需要内核转换来运行它，就会收到通知。这将在sq环标志中设置IORING_SQ_TASKRUN。对于COOP_TASKRUN无效。
#define IORING_SETUP_TASKRUN_FLAG	(1U << 9)

#define IORING_SETUP_SQE128		(1U << 10) /* SQEs are 128 byte */
#define IORING_SETUP_CQE32		(1U << 11) /* CQEs are 32 byte */

enum io_uring_op {
	IORING_OP_NOP,
	IORING_OP_READV,
	IORING_OP_WRITEV,
	IORING_OP_FSYNC,
	IORING_OP_READ_FIXED,
	IORING_OP_WRITE_FIXED,
	IORING_OP_POLL_ADD,
	IORING_OP_POLL_REMOVE,
	IORING_OP_SYNC_FILE_RANGE,
	IORING_OP_SENDMSG,
	IORING_OP_RECVMSG,
	IORING_OP_TIMEOUT,
	IORING_OP_TIMEOUT_REMOVE,
	IORING_OP_ACCEPT,
	IORING_OP_ASYNC_CANCEL,
	IORING_OP_LINK_TIMEOUT,
	IORING_OP_CONNECT,
	IORING_OP_FALLOCATE,
	IORING_OP_OPENAT,
	IORING_OP_CLOSE,
	IORING_OP_FILES_UPDATE,
	IORING_OP_STATX,
	IORING_OP_READ,
	IORING_OP_WRITE,
	IORING_OP_FADVISE,
	IORING_OP_MADVISE,
	IORING_OP_SEND,
	IORING_OP_RECV,
	IORING_OP_OPENAT2,
	IORING_OP_EPOLL_CTL,
	IORING_OP_SPLICE,
	IORING_OP_PROVIDE_BUFFERS,
	IORING_OP_REMOVE_BUFFERS,
	IORING_OP_TEE,
	IORING_OP_SHUTDOWN,
	IORING_OP_RENAMEAT,
	IORING_OP_UNLINKAT,
	IORING_OP_MKDIRAT,
	IORING_OP_SYMLINKAT,
	IORING_OP_LINKAT,
	IORING_OP_MSG_RING,
	IORING_OP_FSETXATTR,
	IORING_OP_SETXATTR,
	IORING_OP_FGETXATTR,
	IORING_OP_GETXATTR,
	IORING_OP_SOCKET,
	IORING_OP_URING_CMD,

	/* this goes last, obviously 显然，这是最后一个 */
	IORING_OP_LAST,
};

/*
 * sqe->fsync_flags
 */
#define IORING_FSYNC_DATASYNC	(1U << 0)

/*
 * sqe->timeout_flags
 */
#define IORING_TIMEOUT_ABS		(1U << 0)
#define IORING_TIMEOUT_UPDATE		(1U << 1)
#define IORING_TIMEOUT_BOOTTIME		(1U << 2)
#define IORING_TIMEOUT_REALTIME		(1U << 3)
#define IORING_LINK_TIMEOUT_UPDATE	(1U << 4)
#define IORING_TIMEOUT_ETIME_SUCCESS	(1U << 5)
#define IORING_TIMEOUT_CLOCK_MASK	(IORING_TIMEOUT_BOOTTIME | IORING_TIMEOUT_REALTIME)
#define IORING_TIMEOUT_UPDATE_MASK	(IORING_TIMEOUT_UPDATE | IORING_LINK_TIMEOUT_UPDATE)
/*
 * sqe->splice_flags
 * extends splice(2) flags
 */
#define SPLICE_F_FD_IN_FIXED	(1U << 31) /* the last bit of __u32 */

/*
 * POLL_ADD flags. Note that since sqe->poll_events is the flag space, the
 * command flags for POLL_ADD are stored in sqe->len.
 *
 * IORING_POLL_ADD_MULTI	Multishot poll. Sets IORING_CQE_F_MORE if
 *				the poll handler will continue to report
 *				CQEs on behalf of the same SQE.
 *
 * IORING_POLL_UPDATE		Update existing poll request, matching
 *				sqe->addr as the old user_data field.
 */
#define IORING_POLL_ADD_MULTI	(1U << 0)
#define IORING_POLL_UPDATE_EVENTS	(1U << 1)
#define IORING_POLL_UPDATE_USER_DATA	(1U << 2)

/*
 * ASYNC_CANCEL flags.
 *
 * IORING_ASYNC_CANCEL_ALL	Cancel all requests that match the given key
 * IORING_ASYNC_CANCEL_FD	Key off 'fd' for cancelation rather than the
 *				request 'user_data'
 * IORING_ASYNC_CANCEL_ANY	Match any request
 */
#define IORING_ASYNC_CANCEL_ALL	(1U << 0)
#define IORING_ASYNC_CANCEL_FD	(1U << 1)
#define IORING_ASYNC_CANCEL_ANY	(1U << 2)

/*
 * send/sendmsg and recv/recvmsg flags (sqe->ioprio)
 *
 * IORING_RECVSEND_POLL_FIRST	If set, instead of first attempting to send
 *				or receive and arm poll if that yields an
 *				-EAGAIN result, arm poll upfront and skip
 *				the initial transfer attempt.
 */
#define IORING_RECVSEND_POLL_FIRST	(1U << 0)

/*
 * accept flags stored in sqe->ioprio
 */
#define IORING_ACCEPT_MULTISHOT	(1U << 0)

/*
 * IO completion data structure (Completion Queue Entry)
 */
struct io_uring_cqe {
	__u64	user_data;	/* sqe->data submission passed back */
	__s32	res;		/* result code for this event 此事件的结果代码 */
	__u32	flags;

	/*
	 * If the ring is initialized with IORING_SETUP_CQE32, then this field
	 * contains 16-bytes of padding, doubling the size of the CQE.
	 */
	__u64 big_cqe[];
};

/*
 * cqe->flags
 *
 * IORING_CQE_F_BUFFER	If set, the upper 16 bits are the buffer ID 如果设置，则上16位为缓冲区ID
 * IORING_CQE_F_MORE	If set, parent SQE will generate more CQE entries 如果设置，父SQE将生成更多的CQE条目
 * IORING_CQE_F_SOCK_NONEMPTY	If set, more data to read after socket recv 如果设置，则在套接字recv后读取更多数据
 */
#define IORING_CQE_F_BUFFER		(1U << 0)
#define IORING_CQE_F_MORE		(1U << 1)
#define IORING_CQE_F_SOCK_NONEMPTY	(1U << 2)

enum {
	IORING_CQE_BUFFER_SHIFT		= 16,
};

/*
 * Magic offsets for the application to mmap the data it needs Magic偏移量用于应用程序映射所需的数据
 */
#define IORING_OFF_SQ_RING		0ULL
#define IORING_OFF_CQ_RING		0x8000000ULL
#define IORING_OFF_SQES			0x10000000ULL

/*
 * Filled with the offset for mmap(2)
 */
struct io_sqring_offsets {
	__u32 head;
	__u32 tail;
	__u32 ring_mask;
	__u32 ring_entries;
	__u32 flags;
	__u32 dropped;
	__u32 array;
	__u32 resv1;
	__u64 resv2;
};

/*
 * sq_ring->flags
 */
#define IORING_SQ_NEED_WAKEUP	(1U << 0) /* needs io_uring_enter wakeup */
#define IORING_SQ_CQ_OVERFLOW	(1U << 1) /* CQ ring is overflown */
#define IORING_SQ_TASKRUN	(1U << 2) /* task should enter the kernel */

struct io_cqring_offsets {
	__u32 head;
	__u32 tail;
	__u32 ring_mask;
	__u32 ring_entries;
	__u32 overflow;
	__u32 cqes;
	__u32 flags;
	__u32 resv1;
	__u64 resv2;
};

/*
 * cq_ring->flags
 */

/* disable eventfd notifications */
#define IORING_CQ_EVENTFD_DISABLED	(1U << 0)

/*
 * io_uring_enter(2) flags
 */
#define IORING_ENTER_GETEVENTS		(1U << 0)
#define IORING_ENTER_SQ_WAKEUP		(1U << 1)
#define IORING_ENTER_SQ_WAIT		(1U << 2)
#define IORING_ENTER_EXT_ARG		(1U << 3)
#define IORING_ENTER_REGISTERED_RING	(1U << 4)

/*
 * Passed in for io_uring_setup(2). Copied back with updated info on success
 */
struct io_uring_params {
	__u32 sq_entries;
	__u32 cq_entries;
	__u32 flags;
	__u32 sq_thread_cpu;
	__u32 sq_thread_idle;
	__u32 features;
	__u32 wq_fd;
	__u32 resv[3];
	struct io_sqring_offsets sq_off;
	struct io_cqring_offsets cq_off;
};

/*
 * io_uring_params->features flags
 */
#define IORING_FEAT_SINGLE_MMAP		(1U << 0)
#define IORING_FEAT_NODROP		(1U << 1)
#define IORING_FEAT_SUBMIT_STABLE	(1U << 2)
#define IORING_FEAT_RW_CUR_POS		(1U << 3)
#define IORING_FEAT_CUR_PERSONALITY	(1U << 4)
#define IORING_FEAT_FAST_POLL		(1U << 5)
#define IORING_FEAT_POLL_32BITS 	(1U << 6)
#define IORING_FEAT_SQPOLL_NONFIXED	(1U << 7)
#define IORING_FEAT_EXT_ARG		(1U << 8)
#define IORING_FEAT_NATIVE_WORKERS	(1U << 9)
#define IORING_FEAT_RSRC_TAGS		(1U << 10)
#define IORING_FEAT_CQE_SKIP		(1U << 11)
#define IORING_FEAT_LINKED_FILE		(1U << 12)

/*
 * io_uring_register(2) opcodes and arguments
 */
enum {
	IORING_REGISTER_BUFFERS			= 0,
	IORING_UNREGISTER_BUFFERS		= 1,
	IORING_REGISTER_FILES			= 2,
	IORING_UNREGISTER_FILES			= 3,
	IORING_REGISTER_EVENTFD			= 4,
	IORING_UNREGISTER_EVENTFD		= 5,
	IORING_REGISTER_FILES_UPDATE		= 6,
	IORING_REGISTER_EVENTFD_ASYNC		= 7,
	IORING_REGISTER_PROBE			= 8,
	IORING_REGISTER_PERSONALITY		= 9,
	IORING_UNREGISTER_PERSONALITY		= 10,
	IORING_REGISTER_RESTRICTIONS		= 11,
	IORING_REGISTER_ENABLE_RINGS		= 12,

	/* extended with tagging */
	IORING_REGISTER_FILES2			= 13,
	IORING_REGISTER_FILES_UPDATE2		= 14,
	IORING_REGISTER_BUFFERS2		= 15,
	IORING_REGISTER_BUFFERS_UPDATE		= 16,

	/* set/clear io-wq thread affinities */
	IORING_REGISTER_IOWQ_AFF		= 17,
	IORING_UNREGISTER_IOWQ_AFF		= 18,

	/* set/get max number of io-wq workers */
	IORING_REGISTER_IOWQ_MAX_WORKERS	= 19,

	/* register/unregister io_uring fd with the ring */
	IORING_REGISTER_RING_FDS		= 20,
	IORING_UNREGISTER_RING_FDS		= 21,

	/* register ring based provide buffer group */
	IORING_REGISTER_PBUF_RING		= 22,
	IORING_UNREGISTER_PBUF_RING		= 23,

	/* this goes last */
	IORING_REGISTER_LAST
};

/* io-wq worker categories */
enum {
	IO_WQ_BOUND,
	IO_WQ_UNBOUND,
};

/* deprecated, see struct io_uring_rsrc_update */
struct io_uring_files_update {
	__u32 offset;
	__u32 resv;
	__aligned_u64 /* __s32 * */ fds;
};

/*
 * Register a fully sparse file space, rather than pass in an array of all
 * -1 file descriptors.
 */
#define IORING_RSRC_REGISTER_SPARSE	(1U << 0)

struct io_uring_rsrc_register {
	__u32 nr;
	__u32 flags;
	__u64 resv2;
	__aligned_u64 data;
	__aligned_u64 tags;
};

struct io_uring_rsrc_update {
	__u32 offset;
	__u32 resv;
	__aligned_u64 data;
};

struct io_uring_rsrc_update2 {
	__u32 offset;
	__u32 resv;
	__aligned_u64 data;
	__aligned_u64 tags;
	__u32 nr;
	__u32 resv2;
};

/* Skip updating fd indexes set to this value in the fd table */
#define IORING_REGISTER_FILES_SKIP	(-2)

#define IO_URING_OP_SUPPORTED	(1U << 0)

struct io_uring_probe_op {
	__u8 op;
	__u8 resv;
	__u16 flags;	/* IO_URING_OP_* flags */
	__u32 resv2;
};

struct io_uring_probe {
	__u8 last_op;	/* last opcode supported */
	__u8 ops_len;	/* length of ops[] array below */
	__u16 resv;
	__u32 resv2[3];
	struct io_uring_probe_op ops[0];
};

struct io_uring_restriction {
	__u16 opcode;
	union {
		__u8 register_op; /* IORING_RESTRICTION_REGISTER_OP */
		__u8 sqe_op;      /* IORING_RESTRICTION_SQE_OP */
		__u8 sqe_flags;   /* IORING_RESTRICTION_SQE_FLAGS_* */
	};
	__u8 resv;
	__u32 resv2[3];
};

struct io_uring_buf {
	__u64	addr;
	__u32	len;
	__u16	bid;
	__u16	resv;
};

struct io_uring_buf_ring {
	union {
		/*
		 * To avoid spilling into more pages than we need to, the
		 * ring tail is overlaid with the io_uring_buf->resv field.
		 */
		struct {
			__u64	resv1;
			__u32	resv2;
			__u16	resv3;
			__u16	tail;
		};
		struct io_uring_buf	bufs[0];
	};
};

/* argument for IORING_(UN)REGISTER_PBUF_RING */
struct io_uring_buf_reg {
	__u64	ring_addr;
	__u32	ring_entries;
	__u16	bgid;
	__u16	pad;
	__u64	resv[3];
};

/*
 * io_uring_restriction->opcode values
 */
enum {
	/* Allow an io_uring_register(2) opcode */
	IORING_RESTRICTION_REGISTER_OP		= 0,

	/* Allow an sqe opcode */
	IORING_RESTRICTION_SQE_OP		= 1,

	/* Allow sqe flags */
	IORING_RESTRICTION_SQE_FLAGS_ALLOWED	= 2,

	/* Require sqe flags (these flags must be set on each submission) */
	IORING_RESTRICTION_SQE_FLAGS_REQUIRED	= 3,

	IORING_RESTRICTION_LAST
};

struct io_uring_getevents_arg {
	__u64	sigmask;
	__u32	sigmask_sz;
	__u32	pad;
	__u64	ts;
};

#endif
```
#### 2. /tools/io_uring/liburing.h

```c
#ifndef LIB_URING_H
#define LIB_URING_H

#ifdef __cplusplus
extern "C" {
#endif

#include <sys/uio.h>
#include <signal.h>
#include <string.h>
#include "../../include/uapi/linux/io_uring.h"
#include <inttypes.h>
#include <linux/swab.h>
#include "barrier.h"

/*
 * Library interface to io_uring
 */
struct io_uring_sq {
	unsigned *khead;
	unsigned *ktail;
	unsigned *kring_mask;
	unsigned *kring_entries;
	unsigned *kflags;
	unsigned *kdropped;
	unsigned *array;
	struct io_uring_sqe *sqes;

	unsigned sqe_head;
	unsigned sqe_tail;

	size_t ring_sz;
};

struct io_uring_cq {
	unsigned *khead;
	unsigned *ktail;
	unsigned *kring_mask;
	unsigned *kring_entries;
	unsigned *koverflow;
	struct io_uring_cqe *cqes;

	size_t ring_sz;
};

struct io_uring {
	struct io_uring_sq sq;
	struct io_uring_cq cq;
	int ring_fd;
};

// 系统调用
extern int io_uring_setup(unsigned entries, struct io_uring_params *p);
extern int io_uring_enter(int fd, unsigned to_submit,
	unsigned min_complete, unsigned flags, sigset_t *sig);
extern int io_uring_register(int fd, unsigned int opcode, void *arg,
	unsigned int nr_args);

/*
 * Library interface
 */
extern int io_uring_queue_init(unsigned entries, struct io_uring *ring,
	unsigned flags);
extern int io_uring_queue_mmap(int fd, struct io_uring_params *p,
	struct io_uring *ring);
extern void io_uring_queue_exit(struct io_uring *ring);
extern int io_uring_peek_cqe(struct io_uring *ring,
	struct io_uring_cqe **cqe_ptr);
extern int io_uring_wait_cqe(struct io_uring *ring,
	struct io_uring_cqe **cqe_ptr);
extern int io_uring_submit(struct io_uring *ring);
extern struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);

/*
 * Must be called after io_uring_{peek,wait}_cqe() after the cqe has
 * been processed by the application.
 */
static inline void io_uring_cqe_seen(struct io_uring *ring,
				     struct io_uring_cqe *cqe)
{
	if (cqe) {
		struct io_uring_cq *cq = &ring->cq;

		(*cq->khead)++;
		/*
		 * Ensure that the kernel sees our new head, the kernel has
		 * the matching read barrier.
		 */
		write_barrier();
	}
}

/*
 * Command prep helpers
 */
static inline void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *data)
{
	sqe->user_data = (unsigned long) data;
}

static inline void *io_uring_cqe_get_data(struct io_uring_cqe *cqe)
{
	return (void *) (uintptr_t) cqe->user_data;
}

static inline void io_uring_prep_rw(int op, struct io_uring_sqe *sqe, int fd,
				    const void *addr, unsigned len,
				    off_t offset)
{
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode = op;
	sqe->fd = fd;
	sqe->off = offset;
	sqe->addr = (unsigned long) addr;
	sqe->len = len;
}

static inline void io_uring_prep_readv(struct io_uring_sqe *sqe, int fd,
				       const struct iovec *iovecs,
				       unsigned nr_vecs, off_t offset)
{
	io_uring_prep_rw(IORING_OP_READV, sqe, fd, iovecs, nr_vecs, offset);
}

static inline void io_uring_prep_read_fixed(struct io_uring_sqe *sqe, int fd,
					    void *buf, unsigned nbytes,
					    off_t offset)
{
	io_uring_prep_rw(IORING_OP_READ_FIXED, sqe, fd, buf, nbytes, offset);
}

static inline void io_uring_prep_writev(struct io_uring_sqe *sqe, int fd,
					const struct iovec *iovecs,
					unsigned nr_vecs, off_t offset)
{
	io_uring_prep_rw(IORING_OP_WRITEV, sqe, fd, iovecs, nr_vecs, offset);
}

static inline void io_uring_prep_write_fixed(struct io_uring_sqe *sqe, int fd,
					     const void *buf, unsigned nbytes,
					     off_t offset)
{
	io_uring_prep_rw(IORING_OP_WRITE_FIXED, sqe, fd, buf, nbytes, offset);
}

static inline void io_uring_prep_poll_add(struct io_uring_sqe *sqe, int fd,
					  unsigned poll_mask)
{
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode = IORING_OP_POLL_ADD;
	sqe->fd = fd;
#if __BYTE_ORDER == __BIG_ENDIAN
	poll_mask = __swahw32(poll_mask);
#endif
	sqe->poll_events = poll_mask;
}

static inline void io_uring_prep_poll_remove(struct io_uring_sqe *sqe,
					     void *user_data)
{
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode = IORING_OP_POLL_REMOVE;
	sqe->addr = (unsigned long) user_data;
}

static inline void io_uring_prep_fsync(struct io_uring_sqe *sqe, int fd,
				       unsigned fsync_flags)
{
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode = IORING_OP_FSYNC;
	sqe->fd = fd;
	sqe->fsync_flags = fsync_flags;
}

static inline void io_uring_prep_nop(struct io_uring_sqe *sqe)
{
	memset(sqe, 0, sizeof(*sqe));
	sqe->opcode = IORING_OP_NOP;
}

#ifdef __cplusplus
}
#endif

#endif
```

### 2.5.3 例子

#### 1. cat与io_uring

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>
#include <sys/mman.h>
#include <sys/uio.h>
#include <linux/fs.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

// 如果由于缺少下面的头文件而导致编译失败，你的内核可能太老了，不支持io_uring。
#include <linux/io_uring.h>

// 环缓冲区的深度
#define QUEUE_DEPTH 1
#define BLOCK_SZ    1024

// 这是x86专用的，内存屏障
#define read_barrier()  __asm__ __volatile__("":::"memory")
// 1）__asm__用于指示编译器在此插入汇编语句
// 2）__volatile__用于告诉编译器，严禁将此处的汇编语句与其它的语句重组合优化。即：原原本本按原来的样子处理这这里的汇编。
// 3） memory强制gcc编译器假设RAM所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作 废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。
// 4）"":::表示这是个空指令。barrier()不用在此插入一条串行化汇编指令。
#define write_barrier() __asm__ __volatile__("":::"memory")

struct app_io_sq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    unsigned *flags;
    unsigned *array;
};

struct app_io_cq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    struct io_uring_cqe *cqes;
};

struct submitter {
    int ring_fd;
    struct app_io_sq_ring sq_ring;
    struct io_uring_sqe *sqes;
    struct app_io_cq_ring cq_ring;
};

struct file_info {
    off_t file_sz;
    struct iovec iovecs[];      /* Referred by readv/writev */
};

// 在编写这段代码的时候，与io_ering相关的系统调用还不是标准C库的一部分。因此，我们滚动自己的系统调用包装器函数。
int io_uring_setup(unsigned entries, struct io_uring_params *p)
{
    return (int) syscall(__NR_io_uring_setup, entries, p);
}

int io_uring_enter(int ring_fd, unsigned int to_submit,
                          unsigned int min_complete, unsigned int flags)
{
    return (int) syscall(__NR_io_uring_enter, ring_fd, to_submit, min_complete,
                   flags, NULL, 0);
}

// 返回传入其打开文件描述符的文件的大小。正确处理常规文件和块设备。漂亮。
off_t get_file_size(int fd) {
    struct stat st;

    if(fstat(fd, &st) < 0) {
        perror("fstat");
        return -1;
    }
    if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0) {
            perror("ioctl");
            return -1;
        }
        return bytes;
    } else if (S_ISREG(st.st_mode))
        return st.st_size;

    return -1;
}

// Io_uring需要大量的设置，看起来非常繁琐，但并不难理解。由于所有这些样板代码，io_uring的作者创建了liburing，它相对容易使用。
// 但是，您应该花点时间来理解这段代码。知道它是如何工作的总是好的。除了吹牛的权利，它确实给你一种奇怪的极客的平静。
int app_setup_uring(struct submitter *s) {
    struct app_io_sq_ring *sring = &s->sq_ring;
    struct app_io_cq_ring *cring = &s->cq_ring;
    struct io_uring_params p;
    void *sq_ptr, *cq_ptr;

    // 我们需要将io_uring_params结构初始化后传递给io_uring_setup()调用。如果需要，我们可以设置任何标志，但对于本例，我们不这样做。
    memset(&p, 0, sizeof(p));
    s->ring_fd = io_uring_setup(QUEUE_DEPTH, &p);
    if (s->ring_fd < 0) {
        perror("io_uring_setup");
        return 1;
    }

    /*
     * io_uring communication happens via 2 shared kernel-user space ring buffers,
     * which can be jointly mapped with a single mmap() call in recent kernels. 
     * While the completion queue is directly manipulated, the submission queue 
     * has an indirection array in between. We map that in as well.
     * */
    // Io_uring通信是通过2个共享的内核用户空间环缓冲区进行的，在最近的内核中，它们可以通过一个mmap()调用进行联合映射。
    // 虽然完成队列是直接操作的，但提交队列之间有一个间接数组。我们也把它映射进去。
    int sring_sz = p.sq_off.array + p.sq_entries * sizeof(unsigned);
    int cring_sz = p.cq_off.cqes + p.cq_entries * sizeof(struct io_uring_cqe);

    // 在内核版本5.4及更高版本中，可以使用单个mmap()调用映射提交缓冲区和完成缓冲区。
    // 与其检查内核版本，推荐的方法是只检查io_uring_params结构的特征字段，这是一个位掩码。
    // 如果设置了IORING_FEAT_SINGLE_MMAP，那么我们可以取消第二个mmap()调用来映射完成环。
    if (p.features & IORING_FEAT_SINGLE_MMAP) {
        if (cring_sz > sring_sz) {
            sring_sz = cring_sz;
        }
        cring_sz = sring_sz;
    }

    /* Map in the submission and completion queue ring buffers.
     * Older kernels only map in the submission queue, though.
     * */
    // 在提交和完成队列循环缓冲区中进行映射。不过，较旧的内核只在提交队列中映射。
    sq_ptr = mmap(0, sring_sz, PROT_READ | PROT_WRITE,
            MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQ_RING);
    if (sq_ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    if (p.features & IORING_FEAT_SINGLE_MMAP) {
        cq_ptr = sq_ptr;
    } else {
        // 在较老的内核中分别映射完成队列环缓冲区
        cq_ptr = mmap(0, cring_sz, PROT_READ | PROT_WRITE,
                MAP_SHARED | MAP_POPULATE,
                s->ring_fd, IORING_OFF_CQ_RING);
        if (cq_ptr == MAP_FAILED) {
            perror("mmap");
            return 1;
        }
    }
    // 将有用的字段保存在全局app_io_sq_ring结构中，以便以后方便参考
    sring->head = sq_ptr + p.sq_off.head;
    sring->tail = sq_ptr + p.sq_off.tail;
    sring->ring_mask = sq_ptr + p.sq_off.ring_mask;
    sring->ring_entries = sq_ptr + p.sq_off.ring_entries;
    sring->flags = sq_ptr + p.sq_off.flags;
    sring->array = sq_ptr + p.sq_off.array;

    /* Map in the submission queue entries array */
    // 映射：提交队列条目数组
    s->sqes = mmap(0, p.sq_entries * sizeof(struct io_uring_sqe),
            PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQES);
    if (s->sqes == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // 将有用的字段保存在全局app_io_cq_ring结构中，以便以后方便参考
    cring->head = cq_ptr + p.cq_off.head;
    cring->tail = cq_ptr + p.cq_off.tail;
    cring->ring_mask = cq_ptr + p.cq_off.ring_mask;
    cring->ring_entries = cq_ptr + p.cq_off.ring_entries;
    cring->cqes = cq_ptr + p.cq_off.cqes;

    return 0;
}

// 输出长度为len的字符串到stdout。我们在这里使用缓冲输出是为了提高效率，因为我们需要逐字符输出。
void output_to_console(char *buf, int len) {
    while (len--) {
        fputc(*buf++, stdout);
    }
}

// 从完成队列中读取。
// 在这个函数中，我们从完成队列中读取完成事件，获取将包含文件数据的数据缓冲区，并将其打印到控制台。
void read_from_cq(struct submitter *s) {
    struct file_info *fi;
    struct app_io_cq_ring *cring = &s->cq_ring;
    struct io_uring_cqe *cqe;
    unsigned head, reaped = 0;

    head = *cring->head;

    do {
        read_barrier(); // 确保以前的写操作是可见的
        // 记住，这是一个环形缓冲区。如果head == tail，则表示缓冲区为空。
        if (head == *cring->tail)
            break;

        // 获取条目
        cqe = &cring->cqes[head & *s->cq_ring.ring_mask];
        fi = (struct file_info*) cqe->user_data;
        if (cqe->res < 0)
            fprintf(stderr, "Error: %s\n", strerror(abs(cqe->res)));

        int blocks = (int) fi->file_sz / BLOCK_SZ;
        if (fi->file_sz % BLOCK_SZ) blocks++;

        for (int i = 0; i < blocks; i++)
            output_to_console(fi->iovecs[i].iov_base, fi->iovecs[i].iov_len);

        head++; // 我们现在已经消耗了这一项
    } while (1);

    *cring->head = head;
    write_barrier();
}

// 提交到提交队列。
// 在这个函数中，我们向提交队列提交请求。您可以提交多种类型的请求。我们的是readv()请求，它是通过IORING_OP_READV指定的。
int submit_to_sq(char *file_path, struct submitter *s) {
    struct file_info *fi;

    int file_fd = open(file_path, O_RDONLY);
    if (file_fd < 0 ) {
        perror("open");
        return 1;
    }

    struct app_io_sq_ring *sring = &s->sq_ring;
    unsigned index = 0, current_block = 0, tail = 0, next_tail = 0;

    off_t file_sz = get_file_size(file_fd);
    if (file_sz < 0)
        return 1;
    off_t bytes_remaining = file_sz;
    int blocks = (int) file_sz / BLOCK_SZ;
    if (file_sz % BLOCK_SZ) blocks++;

    fi = malloc(sizeof(*fi) + sizeof(struct iovec) * blocks);
    if (!fi) {
        fprintf(stderr, "Unable to allocate memory\n");
        return 1;
    }
    fi->file_sz = file_sz;

    // 对于需要读取的文件的每个块，我们分配一个iovec结构体，该结构体被索引到iveec数组中。
    // 该数组作为提交的一部分传入。如果您不理解这一点，那么您需要查看readv()和writev()系统调用是如何工作的。
    while (bytes_remaining) {
        off_t bytes_to_read = bytes_remaining;
        if (bytes_to_read > BLOCK_SZ)
            bytes_to_read = BLOCK_SZ;

        fi->iovecs[current_block].iov_len = bytes_to_read;

        void *buf;
        // 分配内存，参数3指定长度，参数2指定对齐倍数，参数1返回地址
        if( posix_memalign(&buf, BLOCK_SZ, BLOCK_SZ)) {
            perror("posix_memalign");
            return 1;
        }
        fi->iovecs[current_block].iov_base = buf;

        current_block++;
        bytes_remaining -= bytes_to_read;
    }

    // 将我们的提交队列条目添加到SQE环缓冲区的尾部
    next_tail = tail = *sring->tail;
    next_tail++;
    read_barrier();
    index = tail & *s->sq_ring.ring_mask;
    struct io_uring_sqe *sqe = &s->sqes[index];
    sqe->fd = file_fd;
    sqe->flags = 0;
    sqe->opcode = IORING_OP_READV;
    sqe->addr = (unsigned long) fi->iovecs;
    sqe->len = blocks;
    sqe->off = 0;
    sqe->user_data = (unsigned long long) fi;
    sring->array[index] = index;
    tail = next_tail;

    // 更新尾部，以便内核可以看到它
    if(*sring->tail != tail) {
        *sring->tail = tail;
        write_barrier();
    }

    /*
     * Tell the kernel we have submitted events with the io_uring_enter() system
     * call. We also pass in the IOURING_ENTER_GETEVENTS flag which causes the
     * io_uring_enter() call to wait until min_complete events (the 3rd param)
     * complete.
     * */
    // 告诉内核我们已经通过io_uring_enter()系统调用提交了事件。
    // 我们还传入IOURING_ENTER_GETEVENTS标志，它会导致io_uring_enter()调用等待，直到min_complete事件(第三个参数)完成。
    int ret =  io_uring_enter(s->ring_fd, 1, 1,
            IORING_ENTER_GETEVENTS);
    if(ret < 0) {
        perror("io_uring_enter");
        return 1;
    }

    return 0;
}

int main(int argc, char *argv[]) {
    struct submitter s;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        return 1;
    }

    memset(&s, 0, sizeof(s));

    if(app_setup_uring(&s)) {
        fprintf(stderr, "Unable to setup uring!\n");
        return 1;
    }

    for (int i = 1; i < argc; i++) {
        if(submit_to_sq(argv[i], &s)) {
            fprintf(stderr, "Error reading file\n");
            return 1;
        }
        read_from_cq(&s);
    }

    return 0;
}
```

#### 2. cat与liburing

```c
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <liburing.h>
#include <stdlib.h>

#define QUEUE_DEPTH 1
#define BLOCK_SZ    1024

struct file_info {
    off_t file_sz;
    struct iovec iovecs[];      /* Referred by readv/writev */
};

/*
* Returns the size of the file whose open file descriptor is passed in.
* Properly handles regular file and block devices as well. Pretty.
* */

off_t get_file_size(int fd) {
    struct stat st;

    if(fstat(fd, &st) < 0) {
        perror("fstat");
        return -1;
    }
    if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0) {
            perror("ioctl");
            return -1;
        }
        return bytes;
    } else if (S_ISREG(st.st_mode))
        return st.st_size;

    return -1;
}

/*
 * Output a string of characters of len length to stdout.
 * We use buffered output here to be efficient,
 * since we need to output character-by-character.
 * */
void output_to_console(char *buf, int len) {
    while (len--) {
        fputc(*buf++, stdout);
    }
}

/*
 * Wait for a completion to be available, fetch the data from
 * the readv operation and print it to the console.
 * */

int get_completion_and_print(struct io_uring *ring) {
    struct io_uring_cqe *cqe;
    int ret = io_uring_wait_cqe(ring, &cqe);
    if (ret < 0) {
        perror("io_uring_wait_cqe");
        return 1;
    }
    if (cqe->res < 0) {
        fprintf(stderr, "Async readv failed.\n");
        return 1;
    }
    struct file_info *fi = io_uring_cqe_get_data(cqe);
    int blocks = (int) fi->file_sz / BLOCK_SZ;
    if (fi->file_sz % BLOCK_SZ) blocks++;
    for (int i = 0; i < blocks; i ++)
        output_to_console(fi->iovecs[i].iov_base, fi->iovecs[i].iov_len);

    io_uring_cqe_seen(ring, cqe);
    return 0;
}

/*
 * Submit the readv request via liburing
 * */

int submit_read_request(char *file_path, struct io_uring *ring) {
    int file_fd = open(file_path, O_RDONLY);
    if (file_fd < 0) {
        perror("open");
        return 1;
    }
    off_t file_sz = get_file_size(file_fd);
    off_t bytes_remaining = file_sz;
    off_t offset = 0;
    int current_block = 0;
    int blocks = (int) file_sz / BLOCK_SZ;
    if (file_sz % BLOCK_SZ) blocks++;
    struct file_info *fi = malloc(sizeof(*fi) +
                                          (sizeof(struct iovec) * blocks));

    /*
     * For each block of the file we need to read, we allocate an iovec struct
     * which is indexed into the iovecs array. This array is passed in as part
     * of the submission. If you don't understand this, then you need to look
     * up how the readv() and writev() system calls work.
     * */
    while (bytes_remaining) {
        off_t bytes_to_read = bytes_remaining;
        if (bytes_to_read > BLOCK_SZ)
            bytes_to_read = BLOCK_SZ;

        offset += bytes_to_read;
        fi->iovecs[current_block].iov_len = bytes_to_read;

        void *buf;
        if( posix_memalign(&buf, BLOCK_SZ, BLOCK_SZ)) {
            perror("posix_memalign");
            return 1;
        }
        fi->iovecs[current_block].iov_base = buf;

        current_block++;
        bytes_remaining -= bytes_to_read;
    }
    fi->file_sz = file_sz;

    /* Get an SQE */
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    /* Setup a readv operation */
    io_uring_prep_readv(sqe, file_fd, fi->iovecs, blocks, 0);
    /* Set user data */
    io_uring_sqe_set_data(sqe, fi);
    /* Finally, submit the request */
    io_uring_submit(ring);

    return 0;
}

int main(int argc, char *argv[]) {
    struct io_uring ring;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s [file name] <[file name] ...>\n",
                argv[0]);
        return 1;
    }

    /* Initialize io_uring */
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);

    for (int i = 1; i < argc; i++) {
        int ret = submit_read_request(argv[i], &ring);
        if (ret) {
            fprintf(stderr, "Error reading file: %s\n", argv[i]);
            return 1;
        }
        get_completion_and_print(&ring);
    }

    /* Call the clean-up function. */
    io_uring_queue_exit(&ring);
    return 0;
}
```

#### 3. cp和liburing

```c
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <assert.h>
#include <errno.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <liburing.h>

// queue depth
#define QD  2
#define BS (16 * 1024)

static int infd, outfd;

struct io_data {
    int read;
    off_t first_offset, offset;
    size_t first_len;
    struct iovec iov;
};

// 初始化ring
static int setup_context(unsigned entries, struct io_uring *ring) {
    int ret;

    ret = io_uring_queue_init(entries, ring, 0);
    if( ret < 0) {
        fprintf(stderr, "queue_init: %s\n", strerror(-ret));
        return -1;
    }

    return 0;
}

static int get_file_size(int fd, off_t *size) {
    struct stat st;

    if (fstat(fd, &st) < 0 )
        return -1;
    if(S_ISREG(st.st_mode)) {
        *size = st.st_size;
        return 0;
    } else if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;

        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0)
            return -1;

        *size = bytes;
        return 0;
    }
    return -1;
}

static void queue_prepped(struct io_uring *ring, struct io_data *data) {
    struct io_uring_sqe *sqe;

    sqe = io_uring_get_sqe(ring);
    assert(sqe);

    if (data->read)
        io_uring_prep_readv(sqe, infd, &data->iov, 1, data->offset);
    else
        io_uring_prep_writev(sqe, outfd, &data->iov, 1, data->offset);

    io_uring_sqe_set_data(sqe, data);
}

// 排队读
static int queue_read(struct io_uring *ring, off_t size, off_t offset) {
    struct io_uring_sqe *sqe;
    struct io_data *data;

    data = malloc(size + sizeof(*data));
    if (!data)
        return 1;

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        free(data);
        return 1;
    }

    data->read = 1;
    data->offset = data->first_offset = offset;

    data->iov.iov_base = data + 1;
    data->iov.iov_len = size;
    data->first_len = size;

    io_uring_prep_readv(sqe, infd, &data->iov, 1, offset);
    io_uring_sqe_set_data(sqe, data);
    return 0;
}

// 排队写
static void queue_write(struct io_uring *ring, struct io_data *data) {
    data->read = 0;
    data->offset = data->first_offset;

    data->iov.iov_base = data + 1;
    data->iov.iov_len = data->first_len;

    queue_prepped(ring, data);
    io_uring_submit(ring);
}

int copy_file(struct io_uring *ring, off_t insize) {
    unsigned long reads, writes;
    struct io_uring_cqe *cqe;
    off_t write_left, offset;
    int ret;

    write_left = insize;
    writes = reads = offset = 0; // 读写任务数量，文件偏移

    while (insize || write_left) {
        int had_reads, got_comp;

        // 尽可能多地排队读取
        had_reads = reads;
        while (insize) {
            off_t this_size = insize;

            if (reads + writes >= QD)
                break;
            if (this_size > BS)
                this_size = BS;
            else if (!this_size)
                break;

            if (queue_read(ring, this_size, offset))
                break;

            insize -= this_size;
            offset += this_size;
            reads++;
        }

        // 读提交
        if (had_reads != reads) {
            ret = io_uring_submit(ring);
            if (ret < 0) {
                fprintf(stderr, "io_uring_submit: %s\n", strerror(-ret));
                break;
            }
        }

        // 此时队列已满。让我们至少找到一个完成
        got_comp = 0;
        while (write_left) {
            struct io_data *data;

            if (!got_comp) {
                ret = io_uring_wait_cqe(ring, &cqe); // 等待io_uring完成事件
                got_comp = 1;
            } else {
                ret = io_uring_peek_cqe(ring, &cqe); // 检查io_uring完成事件是否可用
                if (ret == -EAGAIN) {
                    cqe = NULL;
                    ret = 0;
                }
            }
            if (ret < 0) {
                fprintf(stderr, "io_uring_peek_cqe: %s\n",
                        strerror(-ret));
                return 1;
            }
            if (!cqe)
                break;

            data = io_uring_cqe_get_data(cqe);
            if (cqe->res < 0) {
                if (cqe->res == -EAGAIN) {
                    queue_prepped(ring, data);
                    io_uring_cqe_seen(ring, cqe);
                    continue;
                }
                fprintf(stderr, "cqe failed: %s\n",
                        strerror(-cqe->res));
                return 1;
            } else if (cqe->res != data->iov.iov_len) {
                /* short read/write; adjust and requeue */
                // 读写了一部分，调整后再进行任务排队
                data->iov.iov_base += cqe->res;
                data->iov.iov_len -= cqe->res;
                queue_prepped(ring, data);
                io_uring_cqe_seen(ring, cqe);
                continue;
            }

            // 全部完成。如果只是写，就没有别的事可做了。如果读，则排队相应的写。
            if (data->read) {
                queue_write(ring, data);
                write_left -= data->first_len;
                reads--;
                writes++;
            } else {
                free(data);
                writes--;
            }
            io_uring_cqe_seen(ring, cqe);
        }
    }

    return 0;
}

int main(int argc, char *argv[]) {
    struct io_uring ring;
    off_t insize;
    int ret;

    if (argc < 3) {
        printf("Usage: %s <infile> <outfile>\n", argv[0]);
        return 1;
    }

    infd = open(argv[1], O_RDONLY);
    if (infd < 0) {
        perror("open infile");
        return 1;
    }

    outfd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (outfd < 0) {
        perror("open outfile");
        return 1;
    }

    if (setup_context(QD, &ring))
        return 1;

    if (get_file_size(infd, &insize))
        return 1;

    ret = copy_file(&ring, insize);

    close(infd);
    close(outfd);
    io_uring_queue_exit(&ring);
    return ret;
}
```

#### 4. 一个带有liburing的web服务器

webserver_liburing.c

```c
#include <stdio.h>
#include <netinet/in.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <signal.h>
#include <liburing.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/utsname.h>

#define SERVER_STRING           "Server: zerohttpd/0.1\r\n"
#define DEFAULT_SERVER_PORT     8000
#define QUEUE_DEPTH             256
#define READ_SZ                 8192

#define EVENT_TYPE_ACCEPT       0
#define EVENT_TYPE_READ         1
#define EVENT_TYPE_WRITE        2

#define MIN_KERNEL_VERSION      5
#define MIN_MAJOR_VERSION       5

struct request {
    int event_type;
    int iovec_count;
    int client_socket;
    struct iovec iov[];
};

struct io_uring ring;

// 未实现的内容
const char *unimplemented_content = \
                                "HTTP/1.0 400 Bad Request\r\n"
                                "Content-type: text/html\r\n"
                                "\r\n"
                                "<html>"
                                "<head>"
                                "<title>ZeroHTTPd: Unimplemented</title>"
                                "</head>"
                                "<body>"
                                "<h1>Bad Request (Unimplemented)</h1>"
                                "<p>Your client sent a request ZeroHTTPd did not understand and it is probably not your fault.</p>"
                                "</body>"
                                "</html>";

// 404的内容
const char *http_404_content = \
                                "HTTP/1.0 404 Not Found\r\n"
                                "Content-type: text/html\r\n"
                                "\r\n"
                                "<html>"
                                "<head>"
                                "<title>ZeroHTTPd: Not Found</title>"
                                "</head>"
                                "<body>"
                                "<h1>Not Found (404)</h1>"
                                "<p>Your client is asking for an object that was not found on this server.</p>"
                                "</body>"
                                "</html>";

// 一个函数，打印系统调用和错误详细信息，然后以错误码1退出。非零意味着事情进展不顺利。
void fatal_error(const char *syscall) {
    perror(syscall);
    exit(1);
}

// 检查内核版本是否满足要求
int check_kernel_version() {
    struct utsname buffer;
    char *p;
    long ver[16];
    int i=0;

    if (uname(&buffer) != 0) {
        perror("uname");
        exit(EXIT_FAILURE);
    }

    p = buffer.release;

    while (*p) {
        if (isdigit(*p)) {
            ver[i] = strtol(p, &p, 10);
            i++;
        } else {
            p++;
        }
    }
    printf("Minimum kernel version required is: %d.%d\n",
            MIN_KERNEL_VERSION, MIN_MAJOR_VERSION);
    if (ver[0] >= MIN_KERNEL_VERSION && ver[1] >= MIN_MAJOR_VERSION ) {
        printf("Your kernel version is: %ld.%ld\n", ver[0], ver[1]);
        return 0;
    }
    fprintf(stderr, "Error: your kernel version is: %ld.%ld\n",
                    ver[0], ver[1]);
    return 1;
}

// 检查html文件是否存在
void check_for_index_file() {
    struct stat st;
    int ret = stat("public/index.html", &st);
    if(ret < 0 ) {
        fprintf(stderr, "ZeroHTTPd needs the \"public\" directory to be "
                "present in the current directory.\n");
        fatal_error("stat: public/index.html");
    }
}

// 实用函数：将字符串转换为小写
void strtolower(char *str) {
    for (; *str; ++str)
        *str = (char)tolower(*str);
}

// 帮助函数：使代码看起来更整洁，内存分配
void *zh_malloc(size_t size) {
    void *buf = malloc(size);
    if (!buf) {
        fprintf(stderr, "Fatal error: unable to allocate memory.\n");
        exit(1);
    }
    return buf;
}

// 这个函数负责设置web服务器使用的主监听套接字。
int setup_listening_socket(int port) {
    int sock;
    struct sockaddr_in srv_addr;

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        fatal_error("socket()");

    int enable = 1; // 设置地址重用
    if (setsockopt(sock,
                   SOL_SOCKET, SO_REUSEADDR,
                   &enable, sizeof(int)) < 0)
        fatal_error("setsockopt(SO_REUSEADDR)");


    memset(&srv_addr, 0, sizeof(srv_addr));
    srv_addr.sin_family = AF_INET;
    srv_addr.sin_port = htons(port);
    srv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    /* We bind to a port and turn this socket into a listening
     * socket.
     * */
    if (bind(sock,
             (const struct sockaddr *)&srv_addr,
             sizeof(srv_addr)) < 0)
        fatal_error("bind()");

    if (listen(sock, 10) < 0)
        fatal_error("listen()");

    return (sock);
}

// 添加任务：接受连接
int add_accept_request(int server_socket, struct sockaddr_in *client_addr,
                       socklen_t *client_addr_len) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_accept(sqe, server_socket, (struct sockaddr *) client_addr,
                         client_addr_len, 0);
    struct request *req = malloc(sizeof(*req));
    req->event_type = EVENT_TYPE_ACCEPT;
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);

    return 0;
}

// 添加任务：从客户端读数据
int add_read_request(int client_socket) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    struct request *req = malloc(sizeof(*req) + sizeof(struct iovec));
    req->iov[0].iov_base = malloc(READ_SZ);
    req->iov[0].iov_len = READ_SZ;
    req->event_type = EVENT_TYPE_READ;
    req->client_socket = client_socket;
    memset(req->iov[0].iov_base, 0, READ_SZ);
    /* Linux kernel 5.5 has support for readv, but not for recv() or read() */
    io_uring_prep_readv(sqe, client_socket, &req->iov[0], 1, 0);
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}

// 添加任务：向客户端发送数据
int add_write_request(struct request *req) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    req->event_type = EVENT_TYPE_WRITE;
    io_uring_prep_writev(sqe, req->client_socket, req->iov, req->iovec_count, 0);
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}

void _send_static_string_content(const char *str, int client_socket) {
    struct request *req = zh_malloc(sizeof(*req) + sizeof(struct iovec));
    unsigned long slen = strlen(str);
    req->iovec_count = 1;
    req->client_socket = client_socket;
    req->iov[0].iov_base = zh_malloc(slen);
    req->iov[0].iov_len = slen;
    memcpy(req->iov[0].iov_base, str, slen);
    add_write_request(req);
}

// 当ZeroHTTPd遇到除GET或POST以外的任何其他HTTP方法时，此函数用于通知客户机。
void handle_unimplemented_method(int client_socket) {
    _send_static_string_content(unimplemented_content, client_socket);
}

// 此函数用于在未找到所请求的文件时向客户端发送“HTTP Not Found”代码和消息。
void handle_http_404(int client_socket) {
    _send_static_string_content(http_404_content, client_socket);
}

// 一旦确定了要提供的静态文件，则使用此函数读取该文件，并使用Linux的sendfile()系统调用将其写入客户端套接字。
// 这为我们省去了在内核和用户空间之间来回传输文件缓冲区的麻烦。
void copy_file_contents(char *file_path, off_t file_size, struct iovec *iov)
{
    int fd = open(file_path, O_RDONLY);
    if (fd < 0)
        fatal_error("read");

    char *buf = zh_malloc(file_size);
    /* We should really check for short reads here */
    int ret = read(fd, buf, file_size);
    if (ret < file_size) {
        fprintf(stderr, "Encountered a short read.\n");
    }
    close(fd);

    iov->iov_base = buf;
    iov->iov_len = file_size;
}

// 获取文件扩展名
const char *get_filename_ext(const char *filename) {
    const char *dot = strrchr(filename, '.');
    if (!dot || dot == filename)
        return "";
    return dot + 1;
}

// 发送HTTP 200 OK报头，服务字符串，对于几种类型的文件，它还可以根据文件扩展名发送内容类型。
// 它还发送内容长度标头。最后，它在一行中自己发送一个'\r\n'，表示头文件的结束和任何内容的开始。
void send_headers(const char *path, off_t len, struct iovec *iov) {
    char small_case_path[1024];
    char send_buffer[1024];
    strcpy(small_case_path, path);
    strtolower(small_case_path);

    char *str = "HTTP/1.0 200 OK\r\n";
    unsigned long slen = strlen(str);
    iov[0].iov_base = zh_malloc(slen);
    iov[0].iov_len = slen;
    memcpy(iov[0].iov_base, str, slen);

    slen = strlen(SERVER_STRING);
    iov[1].iov_base = zh_malloc(slen);
    iov[1].iov_len = slen;
    memcpy(iov[1].iov_base, SERVER_STRING, slen);

    /*
     * Check the file extension for certain common types of files
     * on web pages and send the appropriate content-type header.
     * Since extensions can be mixed case like JPG, jpg or Jpg,
     * we turn the extension into lower case before checking.
     * */
    // 检查网页上某些常见文件类型的文件扩展名，并发送适当的内容类型头文件。
    // 由于扩展名可以混合大小写，如JPG、JPG或JPG，我们在检查之前将扩展名转换为小写。
    const char *file_ext = get_filename_ext(small_case_path);
    if (strcmp("jpg", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/jpeg\r\n");
    if (strcmp("jpeg", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/jpeg\r\n");
    if (strcmp("png", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/png\r\n");
    if (strcmp("gif", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/gif\r\n");
    if (strcmp("htm", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/html\r\n");
    if (strcmp("html", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/html\r\n");
    if (strcmp("js", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: application/javascript\r\n");
    if (strcmp("css", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/css\r\n");
    if (strcmp("txt", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/plain\r\n");
    slen = strlen(send_buffer);
    iov[2].iov_base = zh_malloc(slen);
    iov[2].iov_len = slen;
    memcpy(iov[2].iov_base, send_buffer, slen);

    // 发送内容长度的头文件，即本例中的文件大小。
    sprintf(send_buffer, "content-length: %ld\r\n", len);
    slen = strlen(send_buffer);
    iov[3].iov_base = zh_malloc(slen);
    iov[3].iov_len = slen;
    memcpy(iov[3].iov_base, send_buffer, slen);

    // 当浏览器在一行中看到'\r\n'序列时，它就知道没有更多的头文件了。后跟内容。
    strcpy(send_buffer, "\r\n");
    slen = strlen(send_buffer);
    iov[4].iov_base = zh_malloc(slen);
    iov[4].iov_len = slen;
    memcpy(iov[4].iov_base, send_buffer, slen);
}

// 处理get请求
void handle_get_method(char *path, int client_socket) {
    char final_path[1024];

    // 如果路径以斜杠结尾，客户端可能希望索引文件位于该目录中。
    if (path[strlen(path) - 1] == '/') {
        strcpy(final_path, "public");
        strcat(final_path, path);
        strcat(final_path, "index.html");
    }
    else {
        strcpy(final_path, "public");
        strcat(final_path, path);
    }

    // stat()系统调用将为您提供有关文件的信息，如类型(常规文件、目录等)、大小等。
    struct stat path_stat;
    if (stat(final_path, &path_stat) == -1) {
        printf("404 Not Found: %s (%s)\n", final_path, path);
        handle_http_404(client_socket);
    }
    else {
        // 检查这是否是一个普通/常规文件，而不是一个目录或其他文件
        if (S_ISREG(path_stat.st_mode)) {
            struct request *req = zh_malloc(sizeof(*req) + (sizeof(struct iovec) * 6));
            req->iovec_count = 6;
            req->client_socket = client_socket;
            send_headers(final_path, path_stat.st_size, req->iov);
            copy_file_contents(final_path, path_stat.st_size, &req->iov[5]);
            printf("200 %s %ld bytes\n", final_path, path_stat.st_size);
            add_write_request(req);
        }
        else {
            handle_http_404(client_socket);
            printf("404 Not Found: %s\n", final_path);
        }
    }
}

// 此函数查看所使用的方法并调用适当的处理函数。
// 因为我们只实现GET和POST方法，其它方法handle_unimplemented_method()被调用。这将向客户端发送一个错误。
void handle_http_method(char *method_buffer, int client_socket) {
    char *method, *path, *saveptr;

    method = strtok_r(method_buffer, " ", &saveptr);
    strtolower(method);
    path = strtok_r(NULL, " ", &saveptr);

    if (strcmp(method, "get") == 0) {
        handle_get_method(path, client_socket);
    }
    else {
        handle_unimplemented_method(client_socket);
    }
}

int get_line(const char *src, char *dest, int dest_sz) {
    for (int i = 0; i < dest_sz; i++) {
        dest[i] = src[i];
        if (src[i] == '\r' && src[i+1] == '\n') {
            dest[i] = '\0';
            return 0;
        }
    }
    return 1;
}

// 解析从客服端收到的请求
int handle_client_request(struct request *req) {
    char http_request[1024];
    // 获取第一行，这将是请求
    if(get_line(req->iov[0].iov_base, http_request, sizeof(http_request))) {
        fprintf(stderr, "Malformed request\n");
        exit(1);
    }
    handle_http_method(http_request, req->client_socket);
    return 0;
}

void server_loop(int server_socket) {
    struct io_uring_cqe *cqe;
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    add_accept_request(server_socket, &client_addr, &client_addr_len);

    while (1) {
        int ret = io_uring_wait_cqe(&ring, &cqe);
        struct request *req = (struct request *) cqe->user_data;
        if (ret < 0)
            fatal_error("io_uring_wait_cqe");
        if (cqe->res < 0) {
            fprintf(stderr, "Async request failed: %s for event: %d\n",
                    strerror(-cqe->res), req->event_type);
            exit(1);
        }

        switch (req->event_type) {
            case EVENT_TYPE_ACCEPT:
                // 添加下一个接受任务和读任务
                add_accept_request(server_socket, &client_addr, &client_addr_len);
                add_read_request(cqe->res);
                free(req);
                break;
            case EVENT_TYPE_READ:
                if (!cqe->res) {
                    fprintf(stderr, "Empty request!\n");
                    break;
                }
                handle_client_request(req); // 添加写任务，返回网页
                free(req->iov[0].iov_base); // 释放上次的req
                free(req);
                break;
            case EVENT_TYPE_WRITE:
                for (int i = 0; i < req->iovec_count; i++) {
                    free(req->iov[i].iov_base);
                }
                close(req->client_socket);
                free(req);
                break;
        }
        /* Mark this request as processed */
        io_uring_cqe_seen(&ring, cqe);
    }
}

// 信号处理
void sigint_handler(int signo) {
    printf("^C pressed. Shutting down.\n");
    io_uring_queue_exit(&ring);
    exit(0);
}

int main() {
    if (check_kernel_version()) {
        return EXIT_FAILURE;
    }
    check_for_index_file();
    int server_socket = setup_listening_socket(DEFAULT_SERVER_PORT);
    printf("ZeroHTTPd listening on port: %d\n", DEFAULT_SERVER_PORT);

    signal(SIGINT, sigint_handler);
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
    server_loop(server_socket);

    return 0;
}
```

运行：

```shell
$ mkdir build
$ cd build
$ cmake ..
$ cmake --build .
$ cd ..
$ build/webserver_liburing
Minimum kernel version required is: 5.5
Your kernel version is: 5.6
ZeroHTTPd listening on port: 8000
```

#### 5. 探测支持的功能

数组op_strs中的值不完整

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/utsname.h>
#include <liburing.h>
#include <liburing/io_uring.h>

static const char *op_strs[] = {
        "IORING_OP_NOP",
        "IORING_OP_READV",
        "IORING_OP_WRITEV",
        "IORING_OP_FSYNC",
        "IORING_OP_READ_FIXED",
        "IORING_OP_WRITE_FIXED",
        "IORING_OP_POLL_ADD",
        "IORING_OP_POLL_REMOVE",
        "IORING_OP_SYNC_FILE_RANGE",
        "IORING_OP_SENDMSG",
        "IORING_OP_RECVMSG",
        "IORING_OP_TIMEOUT",
        "IORING_OP_TIMEOUT_REMOVE",
        "IORING_OP_ACCEPT",
        "IORING_OP_ASYNC_CANCEL",
        "IORING_OP_LINK_TIMEOUT",
        "IORING_OP_CONNECT",
        "IORING_OP_FALLOCATE",
        "IORING_OP_OPENAT",
        "IORING_OP_CLOSE",
        "IORING_OP_FILES_UPDATE",
        "IORING_OP_STATX",
        "IORING_OP_READ",
        "IORING_OP_WRITE",
        "IORING_OP_FADVISE",
        "IORING_OP_MADVISE",
        "IORING_OP_SEND",
        "IORING_OP_RECV",
        "IORING_OP_OPENAT2",
        "IORING_OP_EPOLL_CTL",
        "IORING_OP_SPLICE",
        "IORING_OP_PROVIDE_BUFFERS",
        "IORING_OP_REMOVE_BUFFERS",
        "IORING_OP_TEE"
};

int main() {
    struct utsname u;
    uname(&u);
    printf("You are running kernel version: %s\n", u.release);
    printf("This program won't work on kernel versions earlier than 5.6\n");
    struct io_uring_probe *probe = io_uring_get_probe();
    printf("Report of your kernel's list of supported io_uring operations:\n");
    for (char i = 0; i < IORING_OP_LAST; i++ ) {
        printf("%s: ", op_strs[i]);
        if(io_uring_opcode_supported(probe, i))
            printf("yes.\n");
        else
            printf("no.\n");

    }
    free(probe);
    return 0;
}
```

#### 6. 链接任务

让任务按顺序完成

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include "liburing.h"

#define FILE_NAME1   "/tmp/io_uring_test.txt"
#define STR         "Hello, io_uring!"
char buff[32];

int link_operations(struct io_uring *ring) {
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;

    int fd = open(FILE_NAME1, O_RDWR | O_TRUNC | O_CREAT, 0644);
    if (fd < 0 ) {
        perror("open");
        return 1;
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_write(sqe, fd, STR, strlen(STR), 0 );
    sqe->flags |= IOSQE_IO_LINK;

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_read(sqe, fd, buff, strlen(STR),0);
    sqe->flags |= IOSQE_IO_LINK;

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_close(sqe, fd);

    io_uring_submit(ring);

    for (int i = 0; i < 3; i++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                                                            strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }
    printf("Buffer contents: %s\n", buff);
}

int main() {
    struct io_uring ring;

    int ret = io_uring_queue_init(8, &ring, 0);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    link_operations(&ring);
    io_uring_queue_exit(&ring);
    return 0;
}
```

#### 7. 固定缓冲区

使用固定缓冲区的思想是这样的：你提供一组用iovec结构数组描述的缓冲区，并使用io_uring_register_buffers()将它们注册到内核。这将导致内核将这些缓冲区映射进来，避免将来在用户空间之间进行拷贝。然后，你可以使用“固定缓冲区”函数，如io_uring_prep_write_fixed()和io_uring_prep_read_fixed()，指定你想使用的缓冲区的索引。

使用索引 0 和 1 中的缓冲区将两个字符串使用两个固定写入操作 `io_uring_prep_write_fixed()` 写入文件。稍后，我们使用两个固定读取操作`io_uring_prep_read_fixed()`读取文件，这次使用缓冲区索引 2 和 3。然后，我们打印这些读取的结果。

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include "liburing.h"

#define BUF_SIZE    512
#define FILE_NAME1   "/tmp/io_uring_test.txt"
#define STR1        "What is this life if, full of care,\n"
#define STR2        "We have no time to stand and stare."

int start_fixed_buffer_ops(struct io_uring *ring) {
    struct iovec iov[4];
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;

    int fd = open(FILE_NAME1, O_RDWR | O_TRUNC | O_CREAT, 0644);
    if (fd < 0 ) {
        perror("open");
        return 1;
    }

    for (int i = 0; i < 4; i++) {
        iov[i].iov_base = malloc(BUF_SIZE);
        iov[i].iov_len = BUF_SIZE;
        memset(iov[i].iov_base, 0, BUF_SIZE);
    }

    int ret = io_uring_register_buffers(ring, iov, 4);
    if(ret) {
        fprintf(stderr, "Error registering buffers: %s", strerror(-ret));
        return 1;
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    int str1_sz = strlen(STR1);
    int str2_sz = strlen(STR2);
    strncpy(iov[0].iov_base, STR1, str1_sz);
    strncpy(iov[1].iov_base, STR2, str2_sz);
    io_uring_prep_write_fixed(sqe, fd, iov[0].iov_base, str1_sz, 0, 0);

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_write_fixed(sqe, fd, iov[1].iov_base, str2_sz, str1_sz, 1);

    io_uring_submit(ring);

    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_read_fixed(sqe, fd, iov[2].iov_base, str1_sz, 0, 2);

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    io_uring_prep_read_fixed(sqe, fd, iov[3].iov_base, str2_sz, str1_sz, 3);

    io_uring_submit(ring);
    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }
    printf("Contents read from file:\n");
    printf("%s%s", iov[2].iov_base, iov[3].iov_base);
}

int main() {
    struct io_uring ring;

    int ret = io_uring_queue_init(8, &ring, 0);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    start_fixed_buffer_ops(&ring);
    io_uring_queue_exit(&ring);
    return 0;
}
```

#### 8. 轮询提交队列

减少系统调用的数量是io_uring的一个主要目标。为此，io_uring允许您提交I/O请求，而无需进行任何系统调用。这是通过io_uring支持的一个特殊的提交队列轮询特性来完成的。在这种模式下，就在你的程序设置了轮询模式之后，io_uring启动一个特殊的内核线程，该内核线程在共享提交队列中轮询你的程序可能添加的条目。这样，你只需要将条目提交到共享队列中，内核线程就可以看到它并获取提交队列条目，而不需要你的程序调用io_uring_enter()系统调用，这通常由liburing来处理。这是在用户空间和内核之间共享队列的好处。

如何使用这种模式?这个想法很简单。你可以通过在io_uring_params结构的flags成员中设置IORING_SETUP_SQPOLL标志来告诉io_uring你想要使用这种模式。如果与程序一起启动的内核线程在一段时间内没有看到任何提交，它将退出，您的程序将需要再次调用io_uring_enter()系统调用来唤醒它。这段时间可以通过`io_uring_params` 结构体的成员 `sq_thread_idle` 进行配置。但是，如果持续提交，内核轮询器线程应该永远不会休眠。

但是，如果需要使用该特性，还需要将其与io_uring_register_files()结合使用。使用这种方法，您可以预先告诉内核关于文件描述符数组的信息。这只是在启动I/O之前打开的文件描述符的常规数组。在提交过程中，不像通常那样传递文件描述符给io_uring_prep_read()或io_uring_prep_write()这样的调用，你需要在SQE的flags字段中设置IOSQE_FIXED_FILE标志，并从之前设置的文件描述符数组中传递文件描述符的索引。

> 注意：
>
> 在使用liburing时，永远不要直接调用io_uring_enter()系统调用。这通常由liuring的io_uring_submit()函数来处理。它自动确定你是否在使用轮询模式，并处理当你的程序需要调用io_uring_enter()，而不必为此烦恼。
>
> 内核的轮询线程可能会占用大量CPU。在使用此功能时需要非常小心。设置一个非常大的sq_thread_idle值将导致内核线程在程序没有提交时继续消耗CPU。如果您确实希望处理大量的I/O，那么使用这个特性是个好主意。即使这样做，最好将轮询器线程的空闲值最多设置为几秒。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <liburing.h>
#include <string.h>

#define BUF_SIZE    512
#define FILE_NAME1  "/tmp/io_uring_sq_test.txt"
#define STR1        "What is this life if, full of care,\n"
#define STR2        "We have no time to stand and stare."

// 打印轮询线程的状态
void print_sq_poll_kernel_thread_status() {

    if (system("ps --ppid 2 | grep io_uring-sq" ) == 0)
        printf("Kernel thread io_uring-sq found running...\n");
    else
        printf("Kernel thread io_uring-sq is not running.\n");
}

int start_fixed_buffer_ops(struct io_uring *ring) {
    int fds[1];
    char buff1[BUF_SIZE];
    char buff2[BUF_SIZE];
    char buff3[BUF_SIZE];
    char buff4[BUF_SIZE];
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int str1_sz = strlen(STR1);
    int str2_sz = strlen(STR2);

    fds[0] = open(FILE_NAME1, O_RDWR | O_TRUNC | O_CREAT, 0644);
    if (fds[0] < 0 ) {
        perror("open");
        return 1;
    }

    memset(buff1, 0, BUF_SIZE);
    memset(buff2, 0, BUF_SIZE);
    memset(buff3, 0, BUF_SIZE);
    memset(buff4, 0, BUF_SIZE);
    strncpy(buff1, STR1, str1_sz);
    strncpy(buff2, STR2, str2_sz);

    int ret = io_uring_register_files(ring, fds, 1);
    if(ret) {
        fprintf(stderr, "Error registering files: %s", strerror(-ret));
        return 1;
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_write(sqe, 0, buff1, str1_sz, 0);
    io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_write(sqe, 0, buff2, str2_sz, str1_sz);
    io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

    io_uring_submit(ring); // 没有进行io_uring_enter()系统调用

    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }

    print_sq_poll_kernel_thread_status();

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_read(sqe, 0, buff3, str1_sz, 0);
    io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_read(sqe, 0, buff4, str2_sz, str1_sz);
    io_uring_sqe_set_flags(sqe, IOSQE_FIXED_FILE);

    io_uring_submit(ring);

    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }
    printf("Contents read from file:\n");
    printf("%s%s", buff3, buff4);
}

int main() {
    struct io_uring ring;
    struct io_uring_params params;

    if (geteuid()) {
        fprintf(stderr, "You need root privileges to run this program.\n");
        return 1;
    }

    print_sq_poll_kernel_thread_status();

    memset(&params, 0, sizeof(params));
    params.flags |= IORING_SETUP_SQPOLL;
    params.sq_thread_idle = 2000;

    int ret = io_uring_queue_init_params(8, &ring, &params);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    start_fixed_buffer_ops(&ring);
    io_uring_queue_exit(&ring);
    return 0;
}
```

#### 9. 注册eventfd

Io_uring能够在完成发生时向eventfd实例发布事件。该功能允许使用poll(2)或epoll(7)进行多路I/O的进程向兴趣列表中添加一个io_uring注册的eventfd实例文件描述符，以便poll(2)或epoll(7)在发生通过io_uring完成操作时通知它们。这允许这些程序忙于处理它们现有的事件循环，而不是在io_uring_wait_cqe()调用中被阻塞。

eventfd.c：

```c
#include <sys/eventfd.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <liburing.h>
#include <fcntl.h>

#define BUFF_SZ   512

char buff[BUFF_SZ + 1];
struct io_uring ring;

void error_exit(char *message) {
    perror(message);
    exit(EXIT_FAILURE);
}

void *listener_thread(void *data) {
    struct io_uring_cqe *cqe;
    int efd = (int) data;
    eventfd_t v;
    printf("%s: Waiting for completion event...\n", __FUNCTION__);

    // 请注意，eventfd_read()是glibc提供的一个库函数。它本质上是在eventfd上调用read。
    int ret = eventfd_read(efd, &v);
    if (ret < 0) error_exit("eventfd_read");

    printf("%s: Got completion event.\n", __FUNCTION__);

    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        fprintf(stderr, "Error waiting for completion: %s\n",
                strerror(-ret));
        return NULL;
    }
    // 现在我们有了CQE，让我们处理它
    if (cqe->res < 0) {
        fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
    }
    printf("Result of the operation: %d\n", cqe->res);
    io_uring_cqe_seen(&ring, cqe);

    printf("Contents read from file:\n%s\n", buff);
    return NULL;
}

int setup_io_uring(int efd) {
    int ret = io_uring_queue_init(8, &ring, 0);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    io_uring_register_eventfd(&ring, efd);
    return 0;
}

int read_file_with_io_uring() {
    struct io_uring_sqe *sqe;

    sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }

    int fd = open("/etc/passwd", O_RDONLY);
    io_uring_prep_read(sqe, fd, buff, BUFF_SZ, 0);
    io_uring_submit(&ring);

    return 0;
}

int main() {
    pthread_t t;
    int efd;

    // 创建eventfd实例
    efd = eventfd(0, 0);
    if (efd < 0)
        error_exit("eventfd");

    // 创建监听线程
    pthread_create(&t, NULL, listener_thread, (void *)efd);

    sleep(2);

    // io_uring初始化和注册eventfd
    setup_io_uring(efd);

    // 使用io_uring启动读操作
    read_file_with_io_uring();

    // 等待监听线程完成
    pthread_join(t, NULL);

    io_uring_queue_exit(&ring);
    return EXIT_SUCCESS;
}
```



# 三. 其它

## 3.1 libevent

## 3.2 libuv

libev ：功能齐全，高性能的时间循环，轻微地仿效libevent，但是不再像libevent一样有局限性，也修复了它的一些bug。

libevent ：事件通知库

libuv ：跨平台异步I/O。 作者：蜜糖的编码学习基地 https://www.bilibili.com/read/cv13175136 出处：bilibili

## 3.3 fio工具：可以测试各种类型的IO

## 3.4 dpdk

## 3.5 rust_echo_bench：可以测试自编程序的IO吞吐量

https://github.com/haraldh/rust_echo_bench