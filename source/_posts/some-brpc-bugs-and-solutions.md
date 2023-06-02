---
title: 使用brpc遇到的一些bug和解决方案
tags:
  - brpc
  - bug
categories:
  - 杂货
date: 2023-06-02 15:22:11
---


这篇文章记录一下在使用[brpc](https://github.com/apache/brpc/)的时候遇到的bug以及解决方案，长期更新。

<!-- more -->

## 流式RPC遇到"pure virtual method called"错误

[流式RPC](https://brpc.apache.org/zh/docs/client/streaming-rpc/)的使用需要继承`brpc::StreamInputHandler`，并且实现它的三个纯虚函数，如下：  

``` C++
class StreamReciver : public brpc::StreamInputHandler {
public:
  int on_received_messages(brpc::StreamId id, 
                           butil::IOBuf *const messages[], 
                           size_t size) {
    // 当接收到消息时，调用该方法处理消息
  }
  void on_idle_timeout(brpc::StreamId id) {
    // 当超时未接收到消息时，调用该方法处理
  }
  void on_closed(brpc::StreamId id) {
    // 关闭stream以后，调用该方法处理
  }
}
```

在有些应用场景中，`StreamReceiver`可能并不长，在完成一次流式传输以后就立刻析构了，代码结构类似于：

``` C++
/* 流式传输前 */

{
  brpc::StreamId sd;
  brpc::StreamOptions stream_options;
  StreamReciver stream_recevier;      // StreamReceiver实例化
  stream_options.handler = &stream_recevier;
  if(brpc::StreamCreate(&sd, cntl, &stream_options) != 0) { 
    ... ... // 错误处理
  }
  stub->StreamMethod(&cntl, &request, &response, NULL);
  
  ...... // 数据传输

  brpc::StreamClose(sd);  // 传输结束，关闭stream
}

/*流式传输后*/
```

这样的代码结构里，`stream_recevier`的生命周期仅限于中括号内部，一旦运行退出中括号，它就被析构了。但是前面有说到`StreamReciver::on_closed`方法会在stream被关闭以后调用，而`brpc::StreamClose(sd)`不会等到`StreamReciver::on_close`被调用才返回。因此，有可能出现`stream_recevier`先被析构，而后`on_close`被调用的情况，此时虚函数表找到的就是`brpc::StreamInputHandler`这个基类的`on_close`了，是一个纯虚函数，从而引发了"pure virtual method called"的错误。  

### 解决方案

首先，理所当然的解决方案当然是避免这种代码结构，可以让`StreamReciver`作为brpc Service的成员变量，生命周期和服务一致。  

但是，有时候不可避免地会出现这种代码结构，例如不同的情况下需要构造不同的`StreamReciver`，在准备建立stream时才构造，完成传输后立刻销毁。我对于这种情况的对策是给`StreamReciver`加一个`close_`的成员变量，并在`on_close`方法内将其置为`true`。关闭stream以后，轮询`close_`，判断`on_close`方法是否已经被调用，确认被调用后再退出。具体如下：

``` C++
class StreamReciver : public brpc::StreamInputHandler {
public:
  int on_received_messages(brpc::StreamId id, 
                           butil::IOBuf *const messages[], 
                           size_t size) {}
  void on_idle_timeout(brpc::StreamId id) {}
  void on_closed(brpc::StreamId id) { close_ = true; }
  bool is_close() { return close_; }

private:
  bool close_ = false;
}
```

``` C++
/* 流式传输前 */

{
  ...... 
  brpc::StreamClose(sd); // 关闭stream
  while(!stream_recevier) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
  }
}

/*流式传输后*/
```

这样就可以确保`on_close`方法在`stream_recevier`被析构前得到调用，避免纯虚函数调用错误。

## 流式RPC遇到"Fail to parse response from xxxx:xxxx by streaming_rpc at client-side"错误

这个问题在brpc的[issue#392](https://github.com/apache/brpc/issues/392)有被讨论到，具体原因不明，可能是普通RPC和流式RPC混用同一个连接导致的。

### 解决方案

目前的解决方案是客户端创建`brpc::Channel`时，将`brpc::ChannelOptions`的`connection_type`设置为`"pooled"`，如下：  

``` C++
brpc::Channel channel;
brpc::ChannelOptions options;
options.connection_type = "pooled";
if(channel.Init(server_ep, &options) != 0) {
  // 错误处理
}
```

按照这样修改代码后，暂时没遇到错误了，但是不确保是不是真的解决了问题。

