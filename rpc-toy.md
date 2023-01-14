## 实现一个玩具类 rpc

> Remote Procedure Call is a software communication protocol that one program can use to request a service from a program located in another computer on a network without having to understand the network's details. RPC is used to call other processes on the remote systems like a local system. A procedure call is also sometimes known as a function call or a subroutine call.


不过多介绍 RPC 的概念。主要从实现角色来看看实现一个简易版本的 RPC 的步骤。


During an RPC, the following steps take place:

The client calls the client stub. The call is a local procedure call with parameters pushed onto the stack in the normal way.
The client stub packs the procedure parameters into a message and makes a system call to send the message. The packing of the procedure parameters is called marshalling.
The client's local OS sends the message from the client machine to the remote server machine.
The server OS passes the incoming packets to the server stub.
The server stub unpacks the parameters -- called unmarshalling -- from the message.
When the server procedure is finished, it returns to the server stub, which marshals the return values into a message. The server stub then hands the message to the transport layer.
The transport layer sends the resulting message back to the client transport layer, which hands the message back to the client stub.
The client stub unmarshalls the return parameters, and execution returns to the caller.


一次完整的 RPC 过程中，会有如下的步骤：
- client 调用 client stub(客户端的本地存根，代表着远程方法)。这是一次本地调用，会将调用参数以普通方法调用的形式入栈
- client stub 打包参数到消息中，这个过程称为 marshal
- 客户端获取对应的远程服务器地址，发起一次系统调用，本地操作系统将消息从客户端机器发送到远程服务的机器
- server 操作系统将进来的消息包传递给 server stub
- server stub 从消息中解包出请求参数（相应地称为 unmarshal），将根据请求找到对应的 server producer 并执行
- server 端方法执行完成后，server stub 将返回值 marshal 到一个消息中，并将消息传递给传输层
- 传输层将生成的消息发送回客户端传输层，客户端传输层将消息返回给 client stub
- client stub unmarshal 响应消息并获取返回的参数，return 给 client 端调用者



从这个过程中，我们可以看到 client stub 与 server stub 的不同处理，以及 marshal 与 unmarshal 的通用处理。因此可以将整个 RPC 拆分成不同的模块来开发与维护。

- server
   - 服务注册
   - 服务提供者
   - server stub
- client
   - 服务发现与负载均衡
   - 服务调用者
   - client stub
- common
   - 序列化与反序列化
   - 解压缩
   - 消息编码

