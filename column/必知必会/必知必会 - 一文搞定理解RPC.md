@[TOC]

# 前言
RPC的概念相信很多软件从业人员或多或少都接触过，从开发到测试都可能需要跟它打交道。
但是对于为什么要用RPC？RPC的优点是什么？RPC是什么原理？它跟HTTP有什么不同？相信并不是每个人都比较熟悉。
那么今天我们就来了解下到底什么是RPC？

# RPC概念
如果你百度过RPC，那么你一定看过RPC的百科介绍。RPC（Remote Procedure Call）即远程过程调用。
说真的百科里讲的也只有概念，没有一点可以帮助你理解RPC的细节内容。所以关于概念这里要多讲讲其它的辅助内容。

## RPC协议
通常我们所说的RPC其实是说的RPC协议，即一种专门为服务间远程调用而设计的一种通用协议。
该协议基于其它已有的传输协议，规定通信方式为C/S架构；并且在代码开发过程中要屏蔽掉底层通信细节，
让开发者像调用本地方法一样，调用远程服务。

## RPC组成
基于RPC协议内容的说明，再来看看RPC的主要组成内容：
- 确定一个已有的传输协议（TCP\UDP\HTTP\Websocket等）
- 一个客户端通信实现模块(即客户端stub)
- 一个服务端通信实现模块(即服务端stub)
- 选择一个RPC内容协议（如：json、xml、protobuf等）

这是网络上的一张RPC架构组成图，正好包含了上述列举的几项内容。
![RPC](https://img-blog.csdnimg.cn/20190312182425864.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9maXZlMy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

这张图中的`client`就是我们要开发的功能代码。如果我们想调用远程服务的话，可以直接编写类似本地方法的代码；如：
```bash
Calculator.add(1, 2)
```
然鹅，这里并不是真的调用了本地的add实现方法，而是调用了客户端stub模块；
而剩下的事情就交给客户端stub，它会负责与服务端的stub进行通信，使用约定的传输协议，内容协议等；
最后完成远程调用并返回结果给client模块。

## RPC协议
RPC协议是一个开放的协议，不像TCP和HTTP一样规定了统一的标准，任意的使用方都只能使用相同的规则。
RPC协议可以有很多种实现，只要通信双方约定好就行了。目前市面上的RPC协议实现就有很多种：
- dubbo
- rmi
- hessian
- webserivice
- http
- thrift
- memcached
- redis
- rest
- jsonrpc
- motan
- yar
- grpc
- restful

这些协议可以适用于不同的业务场景，比如：dubbo协议适合高频的小数据量调用，hessian则适合文件传输，
而jsonrpc、grpc则适合跨语言的应用。这些协议也与TCP等协议类似，都规定了自己的头信息和body部分，
用于约定通信的规则。

## RPC框架 
不使用RPC框架能不能进行RPC的调用呢？答案当然是可以的！那为什么还需要RPC框架呢？因为有了RPC框架
我们在使用RPC调用时，就会更加的方便了。比如：RPC框架会帮助我们做这些事情：
- 客户端stub、服务端stub的实现
- 通信内容的序列化与反序列化实现（json、xml、protobuf）
- 服务的注册与发现
- 服务方负载均衡
- 并发性能调优

有了RPC框架之后，就不需要再单独的为项目开发这些基础功能了，这样开发具有RPC功能的客户端、
服务端都跟开发普通本地模块一样方便。

# RPC的优点
说了这么多，那么RPC到底有什么优点呢？其实讲RPC的优点要结合RPC的使用场景，否则RPC可能就无法体现它的优势。
通常而言RPC的特点如下：
- 调用远程服务像调用本地方法一样方便
- 多种传输协议可以选择
- 为系统提供较强可扩展性、高可用性、维护性

# RPC与HTTP的区别
在上面的RPC协议中，也许你已经发现了有HTTP协议。是的没错！就是HTTP协议。
所以RPC和HTTP本质上是面向不同场景的产物。而RPC也可以基于HTTP协议来实现信息内容的传输。
除此之外，RPC和HTTP还有如下典型的区别：
- RPC可以基于TCP、HTTP、WebStock等作为基础传输协议，而HTTP只能是http协议
- RPC使用二进制来传输信息内容（体积更小），HTTP则使用文本格式
- RPC的二进制序列化效率高，HTTP的文本序列化（如json）则耗时多
- RPC基于TCP时可以网络IO多路复用，HTTP不能很好的支持
- RPC可以很方便支持服务治理，而HTTP则需要单独实现支持

简而言之，就是RPC在远程调用的场景下，比HTTP更高效，更简洁、附加特性更强大，更适合分布式和微服务的场景。


获取更多感兴趣的文章，请扫描如下二维码！
![关注二维码](https://img-blog.csdnimg.cn/20190117103222240.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdmUz,size_16,color_FFFFFF,t_70)
