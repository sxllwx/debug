# 深入理解gRPC-go

说来惭愧，gRPC在我平时的工作中也用的不少，但是说到对其的了解，却只到了使用protoc将pb文件生成代码并实现接口（golang interface）的程度套在gRPC的框架中。也就是俗称的“滚轮子”，最简单的轮子(一次请求一次回复的模型)滚过，感觉和http并无本质区别。只是少了定义router和解析数据结构的工作，还有一个重要的点就是少了前后端开发来来回回对接口的交流成本。最近，再一次使用到了gRPC，并使用了双向流，跑起来的总体感觉很“欢实”。服务跑了7天，不管是稳定性还是资源消耗都非常OK，所以决定深究gRPC实现的具体细节。这次就先从Server走起吧。

## gRPC-go 版本

为了统一版本，我当前所使用为此刻最新的master分支，git commit为 98e4c7ad3eefd5c1a3cd647c004943ffab4f5722 

## 从简单HellowordServer说起

![image-20200408161634983](/Users/scott/Library/Application Support/typora-user-images/image-20200408161634983.png)

最关键的3句分别为:

1. grpc.NewServer()
2. pb.RegisterGreeterServer()
3. s.Serve()

之后会以这三句为线索展开。

## grpc.NewServer()做了什么？我们可以用它做什么？

![image-20200408162650178](/Users/scott/Library/Application Support/typora-user-images/image-20200408162650178.png)

这段简单的代码中最为“骚气”的就是使用函数调用栈结构的特点递归的将链状的拦截器聚合起来的操作[具体函数入口](https://github.com/grpc/grpc-go/blob/98e4c7ad3eefd5c1a3cd647c004943ffab4f5722/server.go#L1174)。 这些拦截器在每次客户端请求Server端时就会被调用。

根据实现手段的差异，可以在通过拦截器实现:

1. 调用耗时统计
2. RateLimiter
3. 等等...

## pb.RegisterGreeterServer

该函数的作用为将protobuf生成的pb文件中所定义的gRPC服务注册到gRPC Server中，手段相对简单，通过反射取到实现接口的真正实例，并将其注册到

![image-20200408170016980](/Users/scott/Library/Application Support/typora-user-images/image-20200408170016980.png)

虽然gRPC.Server 拥有一个GetServiceInfo的方法获取该grpc.Server中已经注册的Services，但是因为字段较少，我仍然建议使用自己的方法对pb.RegisterGreeterServer进行装饰。

此处可以做如下"骚操作":

1. 将gRPC中生成的服务注册到本公司已经搭建的服务注册中心
2. 此处可以作为协议转换的枢纽，比如可在此处将，gRPC协议切换为HTTP/ 1.1协议。



## s.Serve()

由于此处代码量过大，为了方便理解，笔者做了一个关键路径的流程图。

![image-20200422145833457](/Users/scott/Library/Application Support/typora-user-images/image-20200422145833457.png)