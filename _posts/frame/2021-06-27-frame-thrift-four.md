---
title: "frame - thrift(四)"
subtitle: "thrift服务端实现"
layout: post
author: "Wentao Dong"
date: 2021-06-26 21:00:00
catalog: false
header-style: post
header-img: "img/city_night.png"
mermaid: true
tags:
  - Thrift
  - Java
---

愚蠢的人写的代码只有计算机能懂，优秀的程序员写的代码人人都懂。---- Martin Fowler, 1999

本节将简单介绍下TServer的子类型。

#### TServer 的子类型

##### TSimpleServer

同步阻塞处理服务，直接用主线程接收连接、业务处理、数据读写，一次只能接受一个连接，处理完后才能接受下一个连接。

基本只是用来测试用的，相信几乎不会有人使用。

以下结合代码进行简要说明，注意以下代码仅关注核心处理逻辑，与真实代码存在差异。

核心客户端代码展示：

```java
// 创建TTransport
TTransport transport = new TSocket("localhost", 9090); 
// 打开连接
transport.open(); 
// 创建TProtocol
TProtocol protocol = new  TBinaryProtocol(transport); 
// 创建Client
Calculator.Client client = new Calculator.Client(protocol); 
// 执行方法调用
client.ping(); 
// 关闭连接
transport.close(); 
```

核心服务端代码展示-创建服务：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TServerTransport serverTransport = new TServerSocket(9090);
// 创建服务
TServer server = new TSimpleServer(new Args(serverTransport).processor(processor));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-TSimpleServer.serve()：

```java
// 开始监听端口
serverTransport_.listen();
while (!stopped_) {
  // 接受请求
  TTransport client = serverTransport_.accept();
  // 获取处理器
  TProcessor processor = processorFactory_.getProcessor(client);
  // 用于读取客户端发来消息的TTransport，底层依赖TSocket（全双工的）
  TTransport inputTransport = inputTransportFactory_.getTransport(client);
  // 用于给客户端发送消息的TTransport，底层依赖TSocket（全双工的）
  TTransport outputTransport = outputTransportFactory_.getTransport(client);
  // 用于读取客户端发来消息的TProtocol
  TProtocol inputProtocol = inputProtocolFactory_.getProtocol(inputTransport);
  // 用于给客户端发送消息的TProtocol
  TProtocol outputProtocol = outputProtocolFactory_.getProtocol(outputTransport);
  // 循环处理业务请求
  while (true) {
    // 处理一个业务请求
    processor.process(inputProtocol, outputProtocol);
  }
}
```

#### TThreadPoolServer

异步阻塞处理服务，用主线程监听连接，将接收到的连接丢到线程池中进行业务处理、数据读写，相对于TSimpleServer显著提高了处理性能。

虽然实现了异步（接收连接和处理业务）但读写数据还是阻塞的，浪费CPU资源。如果要支持高并发，需要大量的业务处理线程，线程上下文切换将是不容忽略的代价。

核心客户端代码同TSimpleServer

核心服务端代码展示-创建服务：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TServerTransport serverTransport = new TServerSocket(9090);
// 创建服务
TServer server = new TThreadPoolServer(new TThreadPoolServer.Args(serverTransport).processor(processor));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-TThreadPoolServer.execute()：

```java
// 主线程创建并启动selectAccept线程，开始监听端口
while (!stopped_) {
  // 接收连接
  TTransport client = serverTransport_.accept();
  // 丢到线程池中进行处理
  executorService_.execute(new WorkerProcess(client));
}
```

核心服务端源码展示-WorkerProcess.run()：

```java
// 获取处理器
TProcessor processor = processorFactory_.getProcessor(client);
// 用于读取客户端发来消息的TTransport，底层依赖TSocket（全双工的）
TTransport inputTransport = inputTransportFactory_.getTransport(client);
// 用于给客户端发送消息的TTransport，底层依赖TSocket（全双工的）
TTransport outputTransport = outputTransportFactory_.getTransport(client);
// 用于读取客户端发来消息的TProtocol
TProtocol inputProtocol = inputProtocolFactory_.getProtocol(inputTransport);
// 用于给客户端发送消息的TProtocol
TProtocol outputProtocol = outputProtocolFactory_.getProtocol(outputTransport);
// 循环处理业务请求
while (true) {
  // 处理一个业务请求
  processor.process(inputProtocol, outputProtocol);
}
```

从以上源码看出WorkerProcess.run()与TSimpleServer.serve()基本一致

##### TNonblockingServer

同步非阻塞处理服务，启动一个线程接收连接、业务处理、数据读写，可以接受多个连接，但只有一个线程工作。

实现了非阻塞操作，但是业务处理和接收连接是同一个线程负责的，适合业务处理简单快速的场景，当业务处理复杂时性能问题依旧凸显。

核心客户端代码展示：

```java
// 创建TTransport, 注意这里同步阻塞客户端的实现需要使用TFramedTransport，因为服务端同步非阻塞实现使用了TFramedTransport
TTransport transport = new TFramedTransport(new TSocket("localhost", 9090)); 
// 打开连接
transport.open(); 
// 创建TProtocol
TProtocol protocol = new  TBinaryProtocol(transport); 
// 创建Client
Calculator.Client client = new Calculator.Client(protocol); 
// 执行方法调用
client.ping(); 
// 关闭连接
transport.close(); 
```

核心服务端代码展示-创建服务：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TNonblockingServerSocket socket = new TNonblockingServerSocket(9090);
// 创建服务
TNonblockingServer server = new TNonblockingServer(new TNonblockingServer.Args(socket).processor(processor));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-TNonblockingServer.startThreads()：

```java
// 主线程创建并启动selectAccept线程，开始监听端口
selectAcceptThread_ = new SelectAcceptThread((TNonblockingServerTransport)serverTransport_);
selectAcceptThread_.start();
```

核心服务端源码展示-SelectAcceptThread()：

```java
// 创建时会注册接收请求事件，这样当有连接请求过来时，后面调用select函数就可以获取到
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```

核心服务端源码展示-SelectAcceptThread.run()：

```java
while (!stopped_) {
  // 执行select
  select();
  // 改变感兴趣的事件，接收消息并处理完后需要改为
  processInterestChanges();
}
```

核心服务端源码展示-SelectAcceptThread.select()：

```java
// 执行select操作，阻塞，当有连接请求过来时返回
selector.select();
// 遍历发生事件的key
Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
while (!stopped_ && selectedKeys.hasNext()) {
  SelectionKey key = selectedKeys.next();
  selectedKeys.remove();
  // 如果发生了连接事件
  if (key.isAcceptable()) {
    handleAccept();
  // 如果有数据可读事件
  } else if (key.isReadable()) {
    // deal with reads
    handleRead(key);
  // 如果可写数据
  } else if (key.isWritable()) {
    // deal with writes
    handleWrite(key);
  }
}
```

以上运用了IO多路复用技术，IO多路复用就是用一个线程监控一组资源，这里的资源就是socket。handleAccept、handleRead、handleWrite 不是核心这里不再深入

#### THsHaServer

TNonblockingServer 的子类，半异步/半同步服务，这里我觉得也算是异步非阻塞处理服务，异步是由于业务处理是异步的，非阻塞是由于socket读写是非阻塞的。该服务启动一个线程接收连接、数据读写，同时启动线程池进行业务处理。

融合了TNonblockingServer和TThreadPoolServer的优点，性能自然不错。问题在于接收连接和数据读写还是由同一个线程处理的，两者将会相互影响。

核心客户端代码同TNonblockingServer

核心服务端代码展示-创建服务：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TNonblockingServerSocket serverTransport = new TNonblockingServerSocket(9090);
// 创建服务
TServer server = new THsHaServer(new THsHaServer.Args(serverTransport).processor(processor));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-启动selectAccept线程，同TNonblockingServer

核心服务端源码展示-new SelectAcceptThread()，同TNonblockingServer

核心服务端源码展示-SelectAcceptThread.run()，同TNonblockingServer

核心服务端源码展示-SelectAcceptThread.select()，同TNonblockingServer

核心服务端源码展示-THsHaServer.requestInvoke()：

```java
// 将请求数据封装成Runnable
Runnable invocation = getRunnable(frameBuffer);
// 丢到线程池里异步执行
invoker.execute(invocation);
```

#### TThreadedSelectorServer

异步非阻塞处理服务，启动一个线程接收连接，同时启动一组线程读写数据，使用线程池进行业务处理，三个操作互不影响。

在THsHaServer基础上更近一步，性能杠杠的。既然这个实现最好，开发中没有特殊需求建议直接使用这个。

核心客户端代码同TNonblockingServer：

核心服务端代码展示-创建服务：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TNonblockingServerSocket serverTransport = new TNonblockingServerSocket(9090);
// 创建服务
TServer server = new TThreadedSelectorServer(new TThreadedSelectorServer.Args(serverTransport).processor(processor));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-TThreadedSelectorServer.startThreads()：

```java
// 创建并启动一组SelectorThread线程
for (int i = 0; i < args.selectorThreads; ++i) {
  selectorThreads.add(new SelectorThread(args.acceptQueueSizePerThread));
}
// 创建并启动一个AcceptThread线程
acceptThread = new AcceptThread((TNonblockingServerTransport) serverTransport_,
createSelectorThreadLoadBalancer(selectorThreads));
// 启动SelectorThread线程
for (SelectorThread thread : selectorThreads) {
  thread.start();
}
// 启动AcceptThread线程
acceptThread.start();
```

核心服务端源码展示-AcceptThread.select()：

```java
// 等代连接
acceptSelector.select();

// 处理发生的IO事件
Iterator<SelectionKey> selectedKeys = acceptSelector.selectedKeys().iterator();
while (!stopped_ && selectedKeys.hasNext()) {
  SelectionKey key = selectedKeys.next();
  selectedKeys.remove();
  handleAccept();
}
```

核心服务端源码展示-AcceptThread.handleAccept()：

```java
// 接收请求
TNonblockingTransport client = doAccept();
// 选择处理线程，这里会轮询所有SelectorThread，平均分配任务
SelectorThread targetThread = threadChooser.nextThread();
// 如果FAST_ACCEPT，默认是这种方式，这种方式直接将任务加入acceptedQueue，该队列一般是有界阻塞队列
if (args.acceptPolicy == Args.AcceptPolicy.FAST_ACCEPT || invoker == null) {
  doAddAccept(targetThread, client);
// 如果FAIR_ACCEPT，这种方式利用线程池异步执行doAddAccept，避免加入acceptedQueue时阻塞耽误接收新的连接
} else {
  invoker.submit(new Runnable() { public void run() {
                                    doAddAccept(targetThread, client);
                                  }}); 
} 
```

核心服务端源码展示-SelectorThread.processAcceptedConnections()：

```java
// 从acceptedQueue中取出连接
TNonblockingTransport accepted = acceptedQueue.poll();
// 注册事件
registerAccepted(accepted);
```

核心服务端源码展示-SelectorThread.select()：

```java
// 等待事件发生
doSelect();
// 处理发生的IO事件
Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
while (!stopped_ && selectedKeys.hasNext()) {
  SelectionKey key = selectedKeys.next();
  selectedKeys.remove();
  if (key.isReadable()) {
    // 处理读事件，执行数据读取和业务处理
    handleRead(key);
  } else if (key.isWritable()) {
    // 处理写事件
    handleWrite(key);
  }
}
```

核心服务端源码展示-TThreadedSelectorServer.requestInvoke()：

```java
// 将请求数据封装成Runnable
Runnable invocation = getRunnable(frameBuffer);
// 丢到线程池里异步执行
invoker.execute(invocation);
```

#### TSaslNonblockingServer

异步非阻塞处理服务，用一个线程接收连接，用一组线程处理连接(数据读写和状态转换），用一个线程池处理认证，用一个线程池执行业务处理。

相比TThreadedSelectorServer增加了安全模块(身份认证、数据加密)，如果对安全有要求，可以使用这个。

核心客户端代码展示：

```java
// 创建TTransport
TTransport transport = new TSocket("localhost", 9090); 
String mechanism = "CRAM-MD5";
String protocol = null;
String serverName = "ThriftJavaServer";
Map<String, String> props = null;
CallbackHandler cbh = new SimpleCallbackHandler();
transport = new TSaslClientTransport(mechanism, "test1", protocol, serverName, props, cbh, transport);
// 打开连接
transport.open(); 
// 创建TProtocol
TProtocol protocol = new  TBinaryProtocol(transport); 
// 创建Client
Calculator.Client client = new Calculator.Client(protocol); 
// 执行方法调用
client.ping(); 
// 关闭连接
transport.close(); 
```

核心服务端代码展示-SimpleCallbackHandler：

```java
import javax.security.auth.callback.Callback;
import javax.security.auth.callback.CallbackHandler;
import javax.security.auth.callback.NameCallback;
import javax.security.auth.callback.PasswordCallback;
import javax.security.auth.callback.UnsupportedCallbackException;
import javax.security.sasl.AuthorizeCallback;
import javax.security.sasl.RealmCallback;

public class SimpleCallbackHandler implements CallbackHandler {
    @Override
    public void handle(Callback[] callbacks) throws UnsupportedCallbackException {
        NameCallback nc = null;
        PasswordCallback pc = null;
        AuthorizeCallback ac = null;
        for (Callback callback : callbacks) {
            if (callback instanceof AuthorizeCallback) {
                ac = (AuthorizeCallback) callback;
            } else if (callback instanceof PasswordCallback) {
                pc = (PasswordCallback) callback;
            } else if (callback instanceof NameCallback) {
                nc = (NameCallback) callback;
            } else if (callback instanceof RealmCallback) {
                continue; // realm is ignored
            } else {
                throw new UnsupportedCallbackException(callback,
                        "Unrecognized SASL DIGEST-MD5 Callback: " + callback);
            }
        }

        if (nc != null) {
            nc.setName(nc.getDefaultName());
        }

        if (pc != null) {
            pc.setPassword(getPassword(nc.getDefaultName()));
        }

        if (ac != null) {
            ac.setAuthorized(true);
            ac.setAuthorizedID(ac.getAuthorizationID());
        }
    }

    // 此处仅作演示用，密码和用户名设为一致
    private char[] getPassword(String userName) {
        return userName.toCharArray();
    }
}
```

核心服务端代码展示：

```java
// 创建业务处理器，实现Calculator.Iface接口(根据IDL自动生成)。业务处理部分需要我们自己实现
CalculatorHandler handler = new CalculatorHandler();
// 创建处理器，把上面创建的业务处理器传进去即可
Calculator.Processor processor = new Calculator.Processor(handler);
// 创建监听端口
TNonblockingServerSocket serverTransport = new TNonblockingServerSocket(9090);
// 创建服务
String mechanism = "CRAM-MD5";
String protocol = null;
String serverName = "ThriftJavaServer";
Map<String, String> props = null;
CallbackHandler cbh = new SimpleCallbackHandler();
TServer server = new TSaslNonblockingServer(new TSaslNonblockingServer.Args(serverTransport)
              .networkThreads(6).processingThreads(6).saslThreads(6).processor(processor)
              .addSaslMechanism(mechanism, protocol, serverName, props, cbh));
// 开始监听和处理请求
server.serve();
```

核心服务端源码展示-TSaslNonblockingServer()：

```java
// 创建接收请求线程
acceptor = new AcceptorThread((TNonblockingServerSocket) serverTransport_);
// 创建处理连接线程组
networkThreadPool = new NetworkThreadPool(args.networkThreads);
// 创建处理认证线程池
authenticationExecutor = Executors.newFixedThreadPool(args.saslThreads);
// 创建处理业务线程池
processingExecutor = Executors.newFixedThreadPool(args.processingThreads);
// 认证服务工厂
saslServerFactory = args.saslServerFactory;
// 业务处理工厂
saslProcessorFactory = args.saslProcessorFactory;
```

核心服务端源码展示-TSaslNonblockingServer.acceptNewConnection()：

```java
Iterator<SelectionKey> selectedKeyItr = acceptSelector.selectedKeys().iterator();
while (!stopped_ && selectedKeyItr.hasNext()) {
  SelectionKey selected = selectedKeyItr.next();
  selectedKeyItr.remove();
  while (true) {
    // Accept all available connections from the backlog.
    TNonblockingTransport connection = serverTransport.accept();
    // 交给处理连接线程组，轮询线程，均衡分配给每个NetworkThread线程，加入NetworkThread.incomingConnections队列
    networkThreadPool.acceptNewConnection(connection)
  }
}
```

核心服务端源码展示-TSaslNonblockingServer.run()：

```java
while (!stopped_) {
  // 处理连接：从队列中取出，注册到ioSelecter中
  handleIncomingConnections();
  // 处理状态改变：认证/业务处理完成后进入下一步，主要是改变channel关注(感兴趣)的事件
  handleStateChanges();
  // 监控关注(感兴趣)的事件
  select();
  // 处理数据读写事件&将任务提交到认证线程池/数据处理线程池
  handleIO();
}
```

核心服务端源码展示-TSaslNonblockingServer.handleIO()：

```java
// 遍历发生事件的key
Iterator<SelectionKey> selectedKeyItr = ioSelector.selectedKeys().iterator();
while (!stopped_ && selectedKeyItr.hasNext()) {
  SelectionKey selected = selectedKeyItr.next();
  selectedKeyItr.remove();
  NonblockingSaslHandler saslHandler = (NonblockingSaslHandler) selected.attachment();
  // 如果发生了可读事件
  if (selected.isReadable()) {
    saslHandler.handleRead();
  // 如果可写数据
  } else if (selected.isWritable()) {
    saslHandler.handleWrite();
  }
  // 如果当前处理阶段完成
  if (saslHandler.isCurrentPhaseDone()) {
    tryRunNextPhase(saslHandler);
  }
}
```

核心服务端源码展示-TSaslNonblockingServer.tryRunNextPhase()：

```java
// 下一个阶段
Phase nextPhase = saslHandler.getNextPhase();
// 移动到下一个阶段，这里就会修改channel关注(感兴趣)的事件
saslHandler.stepToNextPhase();
// 下个阶段如果是认证或者业务处理阶段则交给线程池异步处理
switch (nextPhase) {
  case EVALUATING_SASL_RESPONSE:
    authenticationExecutor.submit(new Computation(saslHandler));
    break;
  case PROCESSING:
    processingExecutor.submit(new Computation(saslHandler));
    break;
  case CLOSING:
    saslHandler.runCurrentPhase();
    break;
  default: // waiting for next io event for the current state machine
}
```

#### 小结

本章介绍了Thrift各种Server的优缺点，从中可以看出一步步优化的痕迹，从接收连接、业务处理、数据读写三个任务使用一个线程同步处理开始，最后演变成每个任务单独的线程进行处理，包括使用安全模块后增加的认证线程池，这是一个分工的过程。将三个任务解偶加上非阻塞IO和IO多路复用技术，极大的提高了服务的性能。对NIO和IO多路复用感兴趣的不妨找些资料认真看下，这也是高性能处理常用的技术手段。
