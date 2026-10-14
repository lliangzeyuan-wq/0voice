
1. 事件与业务解耦，回调抽象
	1. 给每个 fd 绑定回调函数：listenfd 绑定`accept_cb`；clientfd 可读绑定`recv_cb`，可写绑定`send_cb`。
	2. epoll 主循环只做一件事：**拿到就绪 fd，调用它注册好的回调**，循环本身不写业务。
2. 每个连接有独立的用户态状态结构体 `struct conn_item connlist[]`
3. 分离读事件、写事件，动态修改 epoll 事件
4. 可扩展，方便切换业务
5. 面向连接对象思维

```c
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>

#include <stdio.h>
#include <string.h>
#include <unistd.h>

#include <pthread.h>
#include <sys/poll.h>
#include <sys/epoll.h>

#define BUFFER_LENGTH           1024

typedef int(*RCALLBACK)(int fd);

//listenfd 触发EPOLLIN事件的时候执行
int accept_cb(int fd);

//clientfd 触发EPOLLIN事件的时候执行
int recv_cb(int fd);

//clientfd 触发EPOLLOUT事件的时候执行
int send_cb(int fd);

//event
int set_event(int fd, int event, int flag);

// conn , fd , buffer , callback
struct conn_item
{
    int fd;
    
    //读的buffer
    char rbuffer[BUFFER_LENGTH];
    int rlen;
    //写的buffer
    char wbuffer[BUFFER_LENGTH];
    int wlen;
   
    //共用体 每次只能用一个，占用内存区里面最大的。节省内存
    union 
    {
        RCALLBACK accept_callback;
        RCALLBACK recv_callback;
    } recv_t;
    
    RCALLBACK send_callback;
};


int epfd = 0;


struct conn_item connlist[1024] =
{
    0
};


#if 0


struct reactor
{
    int epfd;
    struct conn_item * connlist;
};


#else

#endif



int set_event(int fd, int event, int flag)
{
    if (flag)
    {
        struct epoll_event ev;
        ev.events = event;
        ev.data.fd = fd;
        epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
    }
    else 
    {
        struct epoll_event ev;
        ev.events = event;
        ev.data.fd = fd;
        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);

    }

}


int accept_cb(int fd)
{
    struct sockaddr_in clientaddr;
    socklen_t len = sizeof(clientaddr);
    int clientfd = accept(fd, (struct sockaddr *) &clientaddr, &len);

    if (clientfd < 0)
    {
        return - 1;
    }


    set_event(clientfd, EPOLLIN, 1);


    connlist[clientfd].fd = clientfd;
    memset(connlist[clientfd].rbuffer, 0, BUFFER_LENGTH);
    connlist[clientfd].rlen = 0;
    memset(connlist[clientfd].wbuffer, 0, BUFFER_LENGTH);
    connlist[clientfd].wlen = 0;
    connlist[clientfd].recv_t.recv_callback = recv_cb;
    connlist[clientfd].send_callback = send_cb;
    return clientfd;
}


int recv_cb(int fd)
{
    char * buffer = connlist[fd].rbuffer;
    int idx = connlist[fd].rlen;


    int count = recv(fd, buffer + idx, BUFFER_LENGTH - idx, 0);

    if (count == 0)
    {
        printf("disconnect\n");

        //两者都要有，做到一一对应，这样才是合格的断开
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
        return - 1;
    }

    connlist[fd].rlen += count;

#if 0 // echo: need to send
    memcpy(connlist[fd].wbuffer, connlist[fd].rbuffer, connlist[fd].rlen);
    connlist[fd].wlen = connlist[fd].rlen;
#else
    http_response(&connlist[fd]);
#endif
    set_event(fd, EPOLLOUT, 0);


    return count;
}


int send_cb(int fd)
{
    char * buffer = connlist[fd].wbuffer;
    int idx = connlist[fd].wlen;

    int count = send(fd, buffer, idx, 0);

    //event
    set_event(fd, EPOLLIN, 0);

    return count;
}




// tcp
int main()
{

    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    
    //
    connlist[sockfd].fd = sockfd;
    connlist[sockfd].recv_t.accept_callback = accept_cb;

    struct sockaddr_in serveraddr;
    memset(&serveraddr, 0, sizeof(struct sockaddr_in));

    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(2048);

    if (-1 == bind(sockfd, (struct sockaddr *) &serveraddr, sizeof(struct sockaddr)))
    {
        perror("bind");
        return - 1;
    }

    listen(sockfd, 10);



    epfd = epoll_create(1);

    set_event(sockfd, EPOLL_CTL_ADD, 1);


    struct epoll_event events[1024] =
    {
        0
    };


    while (1) //mainloop()
    {
        int nready = epoll_wait(epfd, events, 1024, -1);

        int i = 0;

        for (i = 0; i < nready; ++i)
        {
            int connfd = events[i].data.fd;
            if (events[i].events & EPOLLIN)
            {
                int count = connlist[connfd].recv_t.recv_callback(connfd);
                printf("recv count: %d <-- buffer: %s\n", count, connlist[connfd].rbuffer);
            }
            else if (events[i].events & EPOLLOUT)
            {

                //int count = connlist[connfd].send_callback(connfd);
                printf("send --> buffer: %s\n", connlist[connfd].wbuffer);
                int count = connlist[connfd].send_callback(connfd);
               
            }
        }
    }





    //作用 ： 防止程序直接退出
    getchar();

    //close(clientfd);
}


```