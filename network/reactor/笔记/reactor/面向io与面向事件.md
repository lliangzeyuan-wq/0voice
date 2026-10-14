
# 两种写法：**按fd类型分支** vs **按IO事件类型分支**

这是epoll高并发网络服务器的两种代码组织思路，面试高频。

```
// 左边：按fd类型划分（判断是listenfd还是clientfd）
if(listenfd) {
    accept();  // 监听socket，处理新连接
} else { // clientfd，客户端连接socket
    recv();
    send();
}
```


```
// 右边：按事件类型划分（判断是什么IO事件）
if(events & EPOLLIN) {
    // 可读事件
} else if(events & EPOLLOUT) {
    // 可写事件
}
```

## 1. 左边：面向fd（文件描述符）划分

### 逻辑

判断**这个fd是谁**：是监听套接字listenfd，还是客户端连接套接字clientfd。

- `listenfd`：专门用来**接收新客户端连接**，事件永远是`EPOLLIN`，触发就调用`accept()`拿到新clientfd，把新clientfd加入epoll。
- `clientfd`：已经建立好的客户端连接，这个fd上可以发生可读、可写事件，做`recv`收数据、`send`发数据。

✅ 优点：逻辑直观，新手好写。 ❌ 缺点： `clientfd`里面**可读、可写混在一起写**。当业务复杂，发送缓冲区满、需要异步发送（比如大文件、redis回复），读写状态纠缠在一起，代码容易乱。

> 特点：**先看是谁的fd，再做操作**。

## 2. 右边：面向事件划分（event驱动，工业界主流，Nginx/Redis用这套思想）

不关心当前fd是listenfd还是clientfd，**只关心现在触发了什么事件**

- `EPOLLIN`：fd可读（有数据到来 / 新连接到来）
- `EPOLLOUT`：fd可写（内核发送缓冲区有空位，可以send）

> 注意：listenfd的新连接到来，本质也是触发 **EPOLLIN**。 所以在`EPOLLIN`分支内部，**再判断fd是listenfd还是clientfd**：

- 如果是listenfd → acc# 两种写法：**按fd类型分支** vs **按IO事件类型分支**

这是epoll高并发网络服务器的两种代码组织思路，面试高频。

```
// 左边：按fd类型划分（判断是listenfd还是clientfd）
if(listenfd) {
    accept();  // 监听socket，处理新连接
} else { // clientfd，客户端连接socket
    recv();
    send();
}
```

```
// 右边：按事件类型划分（判断是什么IO事件）
if(events & EPOLLIN) {
    // 可读事件
} else if(events & EPOLLOUT) {
    // 可写事件
}
```

## 1. 左边：面向fd（文件描述符）划分  

### 逻辑

判断**这个fd是谁**：是监听套接字listenfd，还是客户端连接套接字clientfd。

- `listenfd`：专门用来**接收新客户端连接**，事件永远是`EPOLLIN`，触发就调用`accept()`拿到新clientfd，把新clientfd加入epoll。
- `clientfd`：已经建立好的客户端连接，这个fd上可以发生可读、可写事件，做`recv`收数据、`send`发数据。

✅ 优点：逻辑直观，新手好写。 ❌ 缺点： `clientfd`里面**可读、可写混在一起写**。当业务复杂，发送缓冲区满、需要异步发送（比如大文件、redis回复），读写状态纠缠在一起，代码容易乱。

> 特点：**先看是谁的fd，再做操作**。

## 2. 右边：面向事件划分（event驱动，工业界主流，Nginx/Redis用这套思想）

不关心当前fd是listenfd还是clientfd，**只关心现在触发了什么事件**

- `EPOLLIN`：fd可读（有数据到来 / 新连接到来）
- `EPOLLOUT`：fd可写（内核发送缓冲区有空位，可以send）

> 注意：listenfd的新连接到来，本质也是触发 **EPOLLIN**。 所以在`EPOLLIN`分支内部，**再判断fd是listenfd还是clientfd**：

- 如果是listenfd → accept
- 如果是clientfd → recv

✅ 优点： **读写事件完全解耦**。读、写逻辑分开写。 比如发送缓冲区满，就把clientfd注册EPOLLOUT，等内核缓冲区有空了自动触发写事件，再send。适合高并发、异步收发，Nginx、Redis都是事件驱动模型。 ❌ 缺点：需要维护每个连接的状态（读缓冲区、写缓冲区），代码稍微复杂。

## 合并完整伪代码（面试标准写法，把两种思路合在一起）

```
int nready = epoll_wait(epfd, events, 1024, -1);
for(int i=0; i<nready; i++){
    int fd = events[i].data.fd;
    if(events[i].events & EPOLLIN)
    {
        if(fd == listenfd){
            // listenfd可读，新连接
            int cfd = accept(fd, ...);
            // 将cfd加入epoll，监听EPOLLIN
        }else{
            // clientfd可读，收数据
            recv(fd, buf, sizeof(buf),0);
        }
    }
    if(events[i].events & EPOLLOUT)
    {
        // clientfd可写，发送数据
        send(fd, buf, len,0);
    }
}
```

## 一句话总结区别

1. **面向fd划分（左图思路）**：先区分套接字类型，listenfd只做accept，clientfd同时recv+send；读写逻辑写在同一个分支，简单项目用。
2. **面向事件划分（右图思路，事件驱动）**：**先区分事件类型**，EPOLLIN处理读，EPOLLOUT处理写；读写分离，高并发框架Nginx/Redis采用。

## 面试常问

> 问：epoll中listenfd触发什么事件？ 答：**EPOLLIN**，有新连接待accept。

> 问：EPOLLOUT什么时候触发？ 答：内核TCP发送缓冲区有空闲空间，可以往里面写数据。平时clientfd默认不注册EPOLLOUT，只有发送缓冲区满、还有剩余数据要发的时候，才临时注册EPOLLOUT，等可写事件回调继续发送。

要不要我再对比一下：LT和ET在这两套代码里分别要注意什么坑？ept
- 如果是clientfd → recv

✅ 优点： **读写事件完全解耦**。读、写逻辑分开写。 比如发送缓冲区满，就把clientfd注册EPOLLOUT，等内核缓冲区有空了自动触发写事件，再send。适合高并发、异步收发，Nginx、Redis都是事件驱动模型。 ❌ 缺点：需要维护每个连接的状态（读缓冲区、写缓冲区），代码稍微复杂。

## 合并完整伪代码（面试标准写法，把两种思路合在一起）

```
int nready = epoll_wait(epfd, events, 1024, -1);
for(int i=0; i<nready; i++){
    int fd = events[i].data.fd;
    if(events[i].events & EPOLLIN)
    {
        if(fd == listenfd){
            // listenfd可读，新连接
            int cfd = accept(fd, ...);
            // 将cfd加入epoll，监听EPOLLIN
        }else{
            // clientfd可读，收数据
            recv(fd, buf, sizeof(buf),0);
        }
    }
    if(events[i].events & EPOLLOUT)
    {
        // clientfd可写，发送数据
        send(fd, buf, len,0);
    }
}
```

## 一句话总结区别

1. **面向fd划分（左图思路）**：先区分套接字类型，listenfd只做accept，clientfd同时recv+send；读写逻辑写在同一个分支，简单项目用。
2. **面向事件划分（右图思路，事件驱动）**：**先区分事件类型**，EPOLLIN处理读，EPOLLOUT处理写；读写分离，高并发框架Nginx/Redis采用。

## 面试常问

> 问：epoll中listenfd触发什么事件？ 答：**EPOLLIN**，有新连接待accept。

> 问：EPOLLOUT什么时候触发？ 答：内核TCP发送缓冲区有空闲空间，可以往里面写数据。平时clientfd默认不注册EPOLLOUT，只有发送缓冲区满、还有剩余数据要发的时候，才临时注册EPOLLOUT，等可写事件回调继续发送。

要不要我再对比一下：LT和ET在这两套代码里分别要注意什么坑？