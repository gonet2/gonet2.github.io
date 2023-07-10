# 欢迎使用
[gonet/2](https://github.com/gonet2)是新一代游戏服务器骨架，基于[go语言](http://golang.org)开发，采用了先进的[http/2](http://http2.github.io/)作为服务器端主要通信协议，以[microservice](http://martinfowler.com/articles/microservices.html)作为主要思想进行架构，采用[docker](https://www.docker.com/)作为服务发布手段。相比第一代[gonet](http://github.com/xtaci/gonet)，基础技术选型更加先进，结构更加清晰易读可扩展。

## 相关文档
1. [INSTALL.md](https://github.com/gonet2/doc/blob/master/INSTALL.md) -- 安装
2. [CICD.md](https://github.com/gonet2/doc/blob/master/CICD.md) -- 持续集成与持续部署

## 为什么用microservice架构?
业务分离是游戏服务器架构的基本思路，通过职能划分，能够更加合理的调配服务器资源。
资源的大分类包括，IO,CPU,MEM,BANDWIDTH, 例如常见的情景：        

    IO: 如: 数据库，文件服务，消耗大量读写        
    CPU: 如: 游戏逻辑，消耗大量指令        
    MEM: 如: 分词，排名，pubsub， 消耗大量内存   
    BANDWIDTH: 内网带宽高，外网带宽低，物理上越接近的，传输速度越高

玩家对每种服务的请求量有**巨大的不同**，比如逻辑请求100次，分词请求1次，所以，没有必要1:1配置资源，通过microservice方式分离服务，可以根据业务使用情况，按需配置服务器资源。当服务容量增长，如果在monolithic的架构上做，即全部服务揉在一起成一个大进程，会严重浪费资源，比如大量内存被极少被使用的模块占用, 更严重的问题是，单台服务器的资源不是无限制的，虽然目前顶级配置的服务器可以安装高达96GB的内存，但也极其昂贵，部署量大的时候，产生的费用也不容小觑。

## 为什么选HTTP/2?
为了把所有的服务串起来，必须满足的条件有：    
1. 支持一元RPC调用 (一般的请求/应答模式，类似于函数调用)      
2. 支持服务器推送（例如pubsub服务，异步通知）        
3. 支持双向流传递 (网关透传设备数据包到后端，后端应答数据经过网关返回到设备)        

我们暂不想自己设计[RPC](https://en.wikipedia.org/wiki/Remote_procedure_call)，一是目前RPC繁多，没必要重新发明轮子，二是作为开源项目，应充分融入社区，利用现有资源。我们发现目前http/2(rfc7540)满足以上所有条件，google推出的[gRPC](http://grpc.io/)就是一个基于http/2的RPC实现，当前架构中，所有的服务(microservice)全部通过gRPC连接在一起。 http/2支持stream multiplex，即可以在同一条TCP连接上，传输多个stream(1:N)，stream概念能够非常直观的1:1映射玩家双向数据流。

(**请特别注意一点:** HTTP/2仅用于服务器各个服务间的内部通信，和客户端的通信是自定义协议，位于：https://github.com/gonet2/tools/tree/master/proto_scripts)

附: HTTP/2 帧封装         

        +-----------------------------------------------+
        |                 Length (24)                   |
        +---------------+---------------+---------------+
        |   Type (8)    |   Flags (8)   |
        +-+-------------+---------------+-------------------------------+
        |R|                 Stream Identifier (31)                      |
        +=+=============================================================+
        |                   Frame Payload (0...)                      ...
        +---------------------------------------------------------------+
    
                              Figure 1: Frame Layout
          
## 基本服务模型 

                 +
                 |
                 |
                 +----> game1
                 |
    agent1+------>
                 |
                 +----> game2
                 |                +
    agent2+------>                +-----> snowflake
                 |                |
                 +----> game3+---->
                 |                |
                 |                +-----> chat
                 ++               |
                                  +-----> rank
                                  +        

使用方式假定为：         

1. 前端用两个部署在不同物理服务器上的agent服务接入，无状态，客户端随机访问任意一台agent接入，比如使用DNS round-robin方式连接。
2. agent和auth配合处理完鉴权等工作后，数据包透传进入game进行逻辑处理。如果有多台game服务器，那么用户需要指定一个映射关系(userid->server_id)，用来将玩家固定联入某台game服务器。
3. game和各个独立service通信，配合处理逻辑。service如果是无状态的，默认采用round-robin方式请求服务，如果是带状态的，则根据标识联入指定服务器。

具体的服务描述以及使用案例，请进入各个目录中阅读。

## 实际项目中怎么使用gonet/2?
clone下来慢慢改，不提供插件接口式的可升级模块，gonet/2只提供关键通路和demo，我不想做一个侵入式的骨架，只想在架构层面提供一个我认为比较优秀的思路并在此基础上努力做到整体最优。

## 模块划分
进入每个服务阅读对应文档      
1. [agent](https://github.com/gonet2/agent): 网关      
2. [game](https://github.com/gonet2/game): 游戏逻辑     
3. [snowflake](https://github.com/gonet2/snowflake): UUID发生器      

## 模块设计约定
1. **零**配置，配置集中化到coordinator(etcd/consul)，即：/etc distributed概念。                    
2. **理论上**，唯一可能需要的配置为ETCD_HOST环境变量，用于指定ETCD地址。          
3. 其他模块特定的参数(SERVER_ID什么的)，也通过环境变量指定，docker能方便的设定。

## 日志分析模型
![loganalyticmodel](http://gonet2.github.io/log.png)

日志分析是通往数据驱动的关键步骤，内容过于庞大，暂留组建图于此，有机会再详谈。

## 基础设施

![design](http://gonet2.github.io/design.png)

术语：

1. coordinator -- zk, etcd这种分布式一致性服务。
2. message backbone -- 服务器件消息总线，通常为pub/sub模式，数据密集。

基础设施是用于支撑整个架构的基石。

## 链接
* [gonet/2 unity 客户端网络库](https://github.com/en/libunity) -- by ycs
* [Gonet2游戏服务器框架解析](http://blog.csdn.net/q26335804/article/category/5726691)  -- by 高
* [grpc,nsq等源码分析](https://github.com/tenywen/share) -- by tenywen
* [Protobuf安装] (http://ivecode.blog.163.com/blog/static/22094902020156225235159/) -- by __IveCode

PS. 感谢热心网友对源码的解读

## 资料
* protobuf: https://github.com/google/protobuf
* protobuf golang plugin: https://github.com/golang/protobuf
* grpc: http://grpc.io
* http/2: http://http2.github.io/
