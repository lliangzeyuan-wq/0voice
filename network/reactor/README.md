# 介绍
- reactor是一个高性能的网络模型
- 本代码实现的reactor是一个简易的，但依然能看到reactor的许多优点
  - 事件与业务解耦，回调抽象
  - 分离读、写事件
  - 方便拓展等

# 编译运行
```
gcc reactor.c -o reactor
```

```
./reactor
```