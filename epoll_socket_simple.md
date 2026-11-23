# 极轻量的 C++ epoll TCP 服务端示例
**概述:** 基于 Linux epoll 与非阻塞 Socket 实现的轻量级 TCP 服务端示例，支持 ET 模式下的连接管理、数据收发与回显，适合学习 C++ 网络编程。

## 核心类设计
### epoll类声明epoll.h
```cpp
#ifndef	__EPOLL_H_
#define __EPOLL_H_

#include <sys/epoll.h>
#include <cassert>

class Epoll
{
public:
	/**
	 默认是ET模式，当状态发生变化的时候才获得通知
	 */
	Epoll(bool et = true);
	~Epoll();

	/**
	 * 生成epoll句柄
     * max_connections epoll服务需要支持的最大连接数
	 */
	void create(int max_connections);

	/**
	 * 添加监听句柄
	 */
	void add(int fd, long long data, __uint32_t event);

	/**
	 * 修改句柄事件
	 */
	void mod(int fd, long long data, __uint32_t event);

	/**
	 * 删除句柄事件
	 */
	void del(int fd, long long data, __uint32_t event);

	/**
	 * 等待事件
	 */
	int wait(int millsecond);

	/**
     * 获取被触发的事件
	 */
	struct epoll_event& get(int i) { assert(_pevs != 0); return _pevs[i]; }

protected:

	/**
     * fd    句柄，在create函数时被赋值
     * data  辅助的数据, 可以后续在epoll_event中获取到
     * event 需要监听的事件
	 * op    EPOLL_CTL_ADD：注册新的fd到epfd中
	 * 		 EPOLL_CTL_MOD：修改已经注册的fd的监听事件
	 * 	     EPOLL_CTL_DEL：从epfd中删除一个fd
	 */
	void ctrl(int fd, long long data, __uint32_t events, int op);

protected:

    int _epoll_fd;

	int	_max_connections;

	struct epoll_event *_pevs;

    bool _et;
};
#endif
```

### epoll类实现epoll.cpp
```cpp
#include "epoll.h"
#include <unistd.h>

Epoll::Epoll(bool et)
{
	_epoll_fd   = -1;
	_pevs       = NULL;
    _et         = et;
	_max_connections = 1024;
}

Epoll::~Epoll()
{
	if(_pevs != NULL)
	{
		delete[] _pevs;
		_pevs = NULL;
	}

	if(_epoll_fd > 0)
	{
		close(_epoll_fd);
	}
}

void Epoll::ctrl(int fd, long long data, __uint32_t events, int op)
{
	struct epoll_event ev;
	ev.data.u64 = data;
    if(_et)
    {
        ev.events   = events | EPOLLET;
    }
    else
    {
        ev.events   = events;
    }

	epoll_ctl(_epoll_fd, op, fd, &ev);
}

void Epoll::create(int max_connections)
{
	_max_connections = max_connections;

	_epoll_fd = epoll_create(_max_connections + 1);

	if(_pevs != NULL)
	{
		delete[] _pevs;
	}

	_pevs = new epoll_event[_max_connections + 1];
}

void Epoll::add(int fd, long long data, __uint32_t event)
{
	ctrl(fd, data, event, EPOLL_CTL_ADD);
}

void Epoll::mod(int fd, long long data, __uint32_t event)
{
	ctrl(fd, data, event, EPOLL_CTL_MOD);
}

void Epoll::del(int fd, long long data, __uint32_t event)
{
	ctrl(fd, data, event, EPOLL_CTL_DEL);
}

int Epoll::wait(int millsecond)
{
	return epoll_wait(_epoll_fd, _pevs, _max_connections + 1, millsecond);
}
```

### Epoller类声明epoller.h
```cpp
#ifndef __EPOLLER__
#define __EPOLLER__

#include "epoll.h"
#include <string>

using namespace std;

class Epoller
{
public:
    Epoller():m_MaxConn(0),m_Listenfd(0),m_Run(false){}
    int start();
    int Init(const string &strIp, unsigned short uListenPort, unsigned int iMaxConn);
    int CreateListener(const char *addr, unsigned short uPort, int socket_type);
    int Accept();
    int Process(epoll_event *pEvent);

    void SetNBlock(int iSocket);
    void mod(int fd);
    void mod_in(int fd);
    void mod_out(int fd);
    void del(int fd);

private:
    int m_MaxConn;
    int m_Listenfd;
    Epoll m_Epoll;
    bool m_Run;
};

#endif
```

### Epoller类实现epoller.cpp
```cpp
#include <sys/socket.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>

#include <iostream>
#include <unordered_map>
#include <string>

#include "epoller.h"

using namespace std;

const int BUFFER_SIZE = 1024;

// 存储每个客户端的发送缓冲区
unordered_map<int, string> sendBuffers;

void Epoller::SetNBlock(int iSocket)
{
    int iFlags;
    iFlags = fcntl(iSocket, F_GETFL, 0);
    iFlags |= O_NONBLOCK;
    iFlags |= O_NDELAY;
    fcntl(iSocket, F_SETFL, iFlags);
};

int Epoller::CreateListener(const char *addr, unsigned short uPort, int socket_type)
{
    struct sockaddr_in address;
    int reuse_addr = 1;
    int flags = 1;
    int listener;
    memset((char *) &address, 0, sizeof(address));

    address.sin_family = AF_INET;
    address.sin_port = uPort;
    if (inet_pton(AF_INET, addr, &address.sin_addr) <= 0)
    {
        return -1;
    }
    listener = socket(AF_INET, socket_type, 0);
    if (listener < 0)
    {
        return -2;
    }

    setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, (void *)(&(reuse_addr)), sizeof(reuse_addr));
    setsockopt(listener, SOL_SOCKET, SO_KEEPALIVE, &flags, sizeof(flags));
    if (0 != bind(listener, (struct sockaddr *) &address, sizeof(address)))
    {
        close(listener);
        return -3;
    }

    SetNBlock(listener);
    if (socket_type == SOCK_STREAM)
    {
        if (listen(listener, 500) < 0)
        {
            return -4;
        }
    }
    return listener;
}

int Epoller::Init(const string &strIp, 
                 unsigned short uListenPort, 
                 unsigned int iMaxConn)
{
    m_MaxConn = iMaxConn;

    m_Listenfd = CreateListener(strIp.c_str(), htons(uListenPort), SOCK_STREAM);
    if (m_Listenfd < 0)
    {
        return -2;
    }

    m_Epoll.create(m_MaxConn);
    m_Epoll.add(m_Listenfd, m_Listenfd, EPOLLIN | EPOLLET);
    m_Run = true;
    cout<<"Epoller Init success!"<<endl;

    return 0;
}


//接受新连接
int Epoller::Accept()
{
    struct sockaddr_in stSockAddr;
    int iSockAddrSize, iNewSocket;

    iSockAddrSize = sizeof(struct sockaddr_in);

    iNewSocket = accept(m_Listenfd, (struct sockaddr *)&stSockAddr, (socklen_t *)&iSockAddrSize);
    if (iNewSocket > 0)
    {
        //超过最大连接数限制
        if (iNewSocket >= m_MaxConn)
        {
            close(iNewSocket);

            /*
            此处修改为返回0,使epoll可以继续处理之前的请求,如返回-1,此处变为死循环,之前的fd将不能继续处理;
            不过这样将导致同一时间到达监听端口的请求后面也不会再被处理(因为ET模式), 为了尽量保证响应m_iMaxConn之内的用户请求
            */ 
            return 0;
        }

        //设置为非阻塞方式
        SetNBlock(iNewSocket);

        //加入到EPOLL监控中
        m_Epoll.add(iNewSocket, iNewSocket, EPOLLIN | EPOLLOUT | EPOLLERR | EPOLLHUP | EPOLLET);

        return iNewSocket;
    }
    else
    {
        if (errno == EAGAIN)
        {
            return 0;
        }

        // 如果errno不为EAGAIN, 如ECONNABORTED, EPROTO, EINTR等, 继续accept
        return -1;
    }
}

//修改监听事件
void Epoller::mod_in(int fd)
{
	m_Epoll.mod(fd, fd, EPOLLIN | EPOLLERR | EPOLLHUP | EPOLLET);
}

void Epoller::mod_out(int fd)
{
	m_Epoll.mod(fd, fd, EPOLLOUT | EPOLLERR | EPOLLHUP | EPOLLET);
}

void Epoller::mod(int fd)
{
	m_Epoll.mod(fd, fd, EPOLLIN | EPOLLOUT | EPOLLERR | EPOLLHUP | EPOLLET);
}

void Epoller::del(int fd)
{
	m_Epoll.del(fd, fd, EPOLLIN | EPOLLOUT | EPOLLERR | EPOLLHUP | EPOLLET);
}

int Epoller::start()
{
    while (!m_Run)
    {
        usleep(1000);
    }
    cout<<"Epoller::start..."<<endl;

    //  time_t tStartTime = Time::currTime();
    while (m_Run)
    {
        int iReady = m_Epoll.wait(1000);
        if (iReady <= 0)
        {
            //usleep(100);
            continue;
        }

        for (int i = 0 ; i < iReady ; ++i)
        {
            epoll_event &ev = m_Epoll.get(i);
            if (ev.data.fd == m_Listenfd)
            {
                //有新的连接请求
                int iRet = 0;
                do
                {
                    iRet = Accept();
                }
                while (iRet != 0);
            }
            else
            {
                Process(&ev);
            }
        }
    }

    close(m_Listenfd);

    return 0;
}



//处理socket事件
int Epoller::Process(epoll_event *pEvent)
{
    char buffer[BUFFER_SIZE];

    int iSocket=pEvent->data.fd;
    //int iSocket = pEvent->data.u32;
    if (pEvent->events & (EPOLLERR | EPOLLHUP))
    {
        int error;
        socklen_t len;
        getsockopt(iSocket, SOL_SOCKET, SO_ERROR, (void *)&error, &len);

        string strErr = strerror(error);
        cout<<"Process error peer colse socket:"<<iSocket<<", errno:"<<error<<",errmsg:"<<strErr<<endl;

        return 0;
    }

    if (pEvent->events & EPOLLIN)
    {
        //TODO
        cout<<"收到输入报文，读数据..."<<endl;
        bool connectionClosed = false;

        while(true) 
        {
            memset(buffer, 0x00, BUFFER_SIZE);
            int n = read(iSocket, buffer, BUFFER_SIZE);
            if (n == -1)
            {
                // EAGAIN或EWOULDBLOCK表示已经读取完所有数据
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    std::cout << "Read all data from connFd " << iSocket << std::endl;
                    break;
                }
                // 其他错误，关闭连接
                perror("read");
                connectionClosed = true;
                break;
            }
            else if (n == 0)
            {
                // 对端关闭连接
                std::cout << "Connection closed by peer on connFd " << iSocket << std::endl;
                connectionClosed = true;
                break;
            }
            else
            {
                // 处理读取到的数据
                std::cout << "Read " << n << " bytes from connFd " << iSocket << ": " << buffer << std::endl;
                // 将数据添加到发送缓冲区
                sendBuffers[iSocket] += buffer;
                // 修改监听事件为EPOLLOUT，以便发送响应
                mod_out(iSocket);
            }
        }
        if (connectionClosed) 
        {
            //删除socket
            close(iSocket);
            del(iSocket);
        }
    }

    if (pEvent->events & EPOLLOUT)
    {
        //TODO
        cout<<"准备写数据..."<<endl;
        std::cout << "EPOLLOUT event on connFd " << iSocket << std::endl;
        // 检查是否有数据需要发送
        if (sendBuffers.find(iSocket) != sendBuffers.end() && !sendBuffers[iSocket].empty())
        {
            std::string& response = sendBuffers[iSocket];
            const char* data = response.c_str();
            size_t dataLen = response.length();
            size_t bytesWritten = 0;

            // 循环写入数据直到写满或返回EAGAIN/EWOULDBLOCK
            while (bytesWritten < dataLen)
            {
                int n = write(iSocket, data + bytesWritten, dataLen - bytesWritten);
                if (n == -1) 
                {
                    // EAGAIN或EWOULDBLOCK表示暂时无法写入更多数据
                    if (errno == EAGAIN || errno == EWOULDBLOCK) {
                        std::cout << "Write buffer full, will retry later" << std::endl;
                        break;
                    }
                    // 其他错误，关闭连接
                    perror("write");
                    close(iSocket);
                    sendBuffers.erase(iSocket);
                    del(iSocket);
                    return -1;
                } else {
                    bytesWritten += n;
                    std::cout << "Written " << n << " bytes to connFd " << iSocket << ", total: " << bytesWritten << std::endl;
                }
            }
            // 如果数据全部写完，从发送缓冲区移除
            if (bytesWritten == dataLen) 
            {
                std::cout << "All data written to connFd " << iSocket << std::endl;
                sendBuffers.erase(iSocket);
                
                // 修改监听事件为EPOLLIN，继续接收数据
                mod_in(iSocket);
            } else {
                // 部分数据已发送，更新缓冲区
                sendBuffers[iSocket] = response.substr(bytesWritten);
            }
        }
        else
        {
            // 没有数据需要发送，修改监听事件为EPOLLIN
            mod_in(iSocket);
        }
    }

    return 0;
}
```
## 验证测试
### 主程序测试main.cpp
```cpp
#include "epoller.h"

int main()
{
    Epoller epoller;
    epoller.Init("127.0.0.1", 8080, 1000);
    epoller.start();

    return 0;
}
```
### 编译
```shell
g++ main.cpp epoll.cpp epoller.cpp -o main
```
### 启动
```shell
./main
```

### 使用nc发起客户端发送数据
```shell
nc 127.0.0.1 8080
```

