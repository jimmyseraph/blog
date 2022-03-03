---
title: GRPC接口测试全通攻略
date: 2022-03-02 13:57:55
author: 六道先生
categories:
- IT
tags: 
- gRPC
- 协议
- 接口
+ description: gRPC是由Google公司开源的一款高性能的远程过程调用(RPC)框架，采用的是Socket（TCP/IP）协议来实现远程调用。
---
# 什么是RPC
RPC的全称叫做`Remote Procedure Call（远程过程调用）`，意思是将远程（非本地）的一个方法，当作本地的一个方法来调用的一种规范。举例来帮助大家理解：
> 小明写了一段代码，假设定义了一个`function`叫做`SayHi`，并在本地实现了`SayHi`的内容，那么小明在自己的代码里调用这个`SayHi`，就叫做本地过程调用，这个`SayHi`我们就可以认为叫做`Procedure`。
那什么是远程呢？
很简单，小王写了一段代码，他的代码也需要去调用`SayHi`所实现的功能，那么小王就面临着下面几个选择：
> - 在本地重写一遍`SayHi`。嗯，好像有点重复劳动的味道，人家写过了，我干嘛还要再写一遍。
> - 通过`import`，将小明的`SayHi`功能导入，作为自己代码的依赖，嗯，是个办法。实际上，项目中还是重复了小明的这段代码（import进来的也是一样的代码，link的时候还是会出现在一起）。

> 那么最佳的方案，就是把小明写的`SayHi`，当作一个远程的过程，通过网络连接来实现调用，这就是RPC。
# 进一步理解RPC
RPC是规范，不是协议，只要能实现调用远程的过程函数，都可以算作是RPC，而使用什么具体的协议来实现RPC，并没有限制。我们用代码来进一步说明。
我们先写一个A项目，假设有如下接口：
```Golang
type Hello interface {
    SayHi(*proto.SayHiRequest) *proto.SayHiReply
}

SayHiRequest和SayHiReply这两个类型定义假设是这样的：
type SayHiRequest struct {
    Name string 
    Age  int32 
}
type SayHiReply struct {
    Code    int32  
    Message string 
}
```
如果在本地实现了这个接口：
```Golang
type LocalHello struct{}

var _ ex.Hello = (*LocalHello)(nil)

func (l *LocalHello) SayHi(*proto.SayHiRequest) *proto.SayHiReply {
    return &proto.SayHiReply{
        Code:    1000,
        Message: "hello",
    }
}
```
那么，这种就可以在本地调用，这并不能称作RPC：
```Golang
func CallSayHi(h ex.Hello, in *proto.SayHiRequest) ([]byte, error) {
    reply := h.SayHi(in)
    return json.Marshal(reply)
}

func CallLocal() {
    r, err := CallSayHi(new(internal.LocalHello), &proto.SayHiRequest{
        Name: "liudao",
        Age:  18,
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(string(r))
}
```
> CallSayHi传入的第一个参数，是接口Hello，在本地调用时，传入SayHi的本地实现LocalHello，实现了本地的方法调用。

现在我们再来创建一个B项目，实现RPC。在B项目中，我们定义好SayHi的实现：
```Golang
func (s *HelloService) SayHi(ctx context.Context, req *pb.SayHiRequest) (*pb.SayHiReply, error) {
    return &pb.SayHiReply{
        Code:    1000,
        Message: fmt.Sprintf("Hi, %s, you are %d years old.", req.Name, req.Age),
    }, nil
}
```
为了能让其他人可以通过远程来调用，我们用HTTP协议来实现RPC，这里借助gin简单实现一下HTTP Server：
```Golang
func StartWebServiceHello() {
    r := gin.Default()
    r.POST("/sayhi", func(c *gin.Context) {
        re := c.Request.Body
        bs, _ := io.ReadAll(re)
        in := new(pb.SayHiRequest)
        json.Unmarshal(bs, in)
        reply, _ := new(HelloService).SayHi(context.Background(), in)
        c.JSON(200, reply)
    })
    r.Run(":8088")
}
```
好了，现在我们在A项目中，可以写一个Hello接口的实现，并不需要真正实现里面的SayHi方法，而是采用远程调用：
```Golang
type HttpHello struct{}

var _ ex.Hello = (*HttpHello)(nil)

func (h *HttpHello) SayHi(request *proto.SayHiRequest) *proto.SayHiReply {
    requestBodyString, _ := json.Marshal(request)
    c := &http.Client{}
    req, _ := http.NewRequest("POST", "http://127.0.0.1:8088/sayhi", strings.NewReader(string(requestBodyString)))
    resp, err := c.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    reply := new(proto.SayHiReply)
    json.Unmarshal(body, reply)
    return reply
}
```
实现中使用了http协议，把参数变为json格式的请求内容，当收到响应后，再把json格式的响应转成方法返回的对象，这就是整个rpc的实现思路：将远程的过程所需要传入的参数，先序列化为可在网络上传输的格式（任何字节流都可以），交给远程处理，当收到响应后，再反序列化为远程过程的出参对象，让本地调用远程的方法，就像调用本地方法一样。<br>
我们看下本地调用远程的方法：
```Golang
func CallHTTP() {
    r, err := CallSayHi(new(http.HttpHello), &proto.SayHiRequest{
        Name: "liudao",
        Age:  18,
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(string(r))
}
```
CallHTTP里面的内容和CallLocal里面的代码几乎一模一样，唯一的差别只是实现类换成了HttpHello。<br>
我们让调用远程的方法就像调用本地的方法一样，这就是RPC，而我们使用的协议是HTTP。
# 理解gRPC
## 什么是gRPC
gRPC是由Google公司开源的一款高性能的远程过程调用(RPC)框架，采用的是Socket（TCP/IP）协议来实现远程调用。
## 为何使用Socket
采用HTTP协议虽然有比较好的可读性，但是HTTP协议本身的各种header、换行符等，都占了很多的字节，导致整个协议的数据包显得比较冗余，而socket协议则可以自己定义应用层数据包，这样完全可以使用二进制字节流编码，使得传输的序列化参数本身所占字节数更小，大大节省了传输的带宽消耗，也提高了性能。
## Protocol Buffers
可以这么说，Protocol Buffers（后面简称PB）是gRPC自定义的socket连接应用层的数据传输格式，当然，PB也提供了一整套的工具，让开发者可以很方便的生成所需的PB数据格式，使得开发者可以更容易的使用gRPC。
### 数据格式文件
PB使用`.proto`文件来定义数据格式，经历了V2版本和V3版本，现在主要是使用V3版本的语法。语法非常简单，大家可以想象，`.proto`文件就类似`.xml`文件，只为了进行数据结构的描述，但是proto文件会更加简单和直观，可读性更强。语法详见[<Protocol Buffers官网>](https://developers.google.com/protocol-buffers/docs/proto3) <br>
我这里只给一个简单的例子：
```ProtoBuf
syntax = "proto3";

package proto;

service Hello {
    rpc SayHi (SayHiRequest) returns (SayHiReply);
}

message SayHiRequest {
    string name = 1;
    int32 age = 2;
}

message SayHiReply {
    int32 code = 1;
    string message = 2;
}
```
> 第一行是语法版本说明，表示使用proto3的语法。
package关键字定义了proto文件所在的package，以此做为namespace。主要用于在被其他proto文件引用时避免重名出现。
service关键字用于定义一个rpc服务的名称，其中的rpc关键字，定义了一个rpc的方法，包括它的入参和出参。
message关键字用于定义参数的数据结构，定义方式使用：变量类型 变量名 = 编号
在数据格式定义中，编号并没有特殊的含义，只要在同一个数据结构中不同就可以了，相当于给变量一个ID，在序列化和反序列化时，更方便定位。

### proto文件编译
Google提供了两类工具，首先是跟编程语言无关的protoc，也就是proto编译器（proto compiler），它可以将proto文本文件编译为descriptor文件，这个文件的后缀为.pb，是一个二进制文件，与编程语言无关，通常用于被自身的API来读取创建对应的desciptor对象。<br>
命令格式为：
```bash
$ protoc -I<proto_path> --descriptor_set_out=<pb_out_path> --include_imports <PROTO_FILES>
```
> -I参数用于指定proto文件所在的根目录，注意这里的根目录，意思为package所在的根目录，而不是proto文件所在的目录。<br>
--descriptor_set_out参数用于指定pb二进制文件的输出位置。<br>
--include_imports参数用于标识需要包含所有的import依赖。<br>
PROTO_FILES就是所有的proto文件，多个文件用空格分开，如果文件太多，可以将文件名写入一个文本文件中，然后用@文件名的方式来代替。<br>

在这个工具的基础上，又提供了各个编程语言的插件，官方目前支持：
- C++
- Golang
- Dart
- Java
- Kotlin
- Python
- Ruby
- C#
- Objective-C
- JavaScript
- PHP

语言插件在protoc命令行通过参数来加载，可以使其编译为对应语言的代码。这里以Golang为例，我们来编译之前的`hello.proto`文件：
```Bash
$ protoc --proto_path=./proto --go_out=paths=source_relative:./proto --go-grpc_out=paths=source_relative:./proto ./proto/hello.proto
```
> --go_out参数是golang插件的参数，用于指定message定义的数据格式的go语言代码文件输出位置，对于hello.proto，则会生成hello.pb.go文件。<br>
--go-grpc_out参数同样是golang插件的参数，用于指定rpc service描述文服务的go语言代码文件输出位置，对于hello.proto，则会生成hello_grpc.pb.go文件。

## 让gRPC服务运行起来
有了hello.pb.go和hello_grpc.pb.go，我们就可以来实现我们在前面做的那个服务了，只是由原来的http协议，变为了gRPC协议。
### 实现Hello服务
```Golang
type HelloService struct {
    pb.UnimplementedHelloServer
}

func (s *HelloService) SayHi(ctx context.Context, req *pb.SayHiRequest) (*pb.SayHiReply, error) {
    return &pb.SayHiReply{
        Code:    1000,
        Message: fmt.Sprintf("Hi, %s, you are %d years old.", req.Name, req.Age),
    }, nil
}

func StartRPCService() {
    rpc := grpc.NewServer()
    pb.RegisterHelloServer(rpc, new(HelloService))
    listener, err := net.Listen("tcp", ":8082")
    if err != nil {
        panic(err)
    }
    _ = rpc.Serve(listener)
}
```
HelloService其实就是实现了hello_grpc.pb.go中的接口：
```Golang
type HelloServer interface {
    SayHi(context.Context, *SayHiRequest) (*SayHiReply, error)
    mustEmbedUnimplementedHelloServer()
}
```
这里我们绑定了8082端口作为gRPC服务的监听端口，然后启动服务。
### 实现客户端的访问
我们依然按照直接的模式，来写客户端：
```Golang
type GRPCHello struct{}

var _ ex.Hello = (*GRPCHello)(nil)

func (h *GRPCHello) SayHi(request *proto.SayHiRequest) *proto.SayHiReply {
    conn, err := g.Dial("localhost:8082", g.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        panic(err)
    }
    client := proto.NewHelloClient(conn)
    reply, err := client.SayHi(context.Background(), request)
    if err != nil {
        panic(err)
    }
    return reply
}
```
> 连接gRPC服务，需要用google.golang.org/grpc包提供的Dial方法，来进行socket连接。这里我们的服务端起的比较简单，没有TLS，所以我们的客户端要忽略安全连接才能正常工作。

客户端同样需要hello.pb.go和hello_grpc.pb.go文件，使用hello_grpc.pb.go提供的newClient方法，获取客户端对象，就可以访问SayHi方法了。<br>
调用依然一样：
```Golang
func CallGRPC() {
    r, err := CallSayHi(new(grpc.GRPCHello), &proto.SayHiRequest{
        Name: "liudao",
        Age:  18,
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(string(r))
}
```
# 如何进行gRPC接口测试
对于gRPC接口，如果能理解上上一章的内容，那么就没有什么神秘可言了。目前，没有什么特别方便的工具，可以直接进行gRPC接口测试，Postman目前也是不支持gRPC接口，所以只能使用自己擅长的编程语言，来进行gRPC接口功能测试。当然，这也是一个直接进行接口自动化测试的好机会。
1. 首先要从开发那里拿到接口对应的proto文件，将文件按照开发同样的目录结构存放好。
2. 使用protoc命令进行编译，根据自己擅长的编程语言，使用合适的插件，将proto文件编译成为对应语言的代码文件。
3. 引入google的grpc库，实现gRPC客户端连接。
4. 同HTTP接口测试一样，设计对应的测试用例。
5. 使用代码实现接口测试用例。

# 如何进行gPRC接口性能测试
推荐使用JMeter的[gRPC插件](https://github.com/zalopay-oss/jmeter-grpc-request/releases)，下载最新版后，将jar包存放到jmeter的lib/ext目录下，重新打开JMeter，就可以看到如下的gRPC取样器：<br>
{% asset_img jmeter.jpg jmeter-grpc-request %}

做一下简单的说明：
> <b>Server Name or IP</b>： gRPC服务的地址；<br>
<b>Proto Root Directory</b>：指定proto文件的根目录，这里的根目录，指的是package根目录，而不是proto文件本身所在目录。假设proto中定义了package是`vip.testops.proto`，而这个proto文件的绝对路径是`/Users/code/project_a/vip/testops/proto/Hello.proto`，那么这里的Root Directory就应该是`/Users/code/project_a`；<br>
<b>Full Method</b>：完整方法名，是以`package名.service名/方法名`的形式展现的，比如`vip.testops.proto.HelloService/SayHi`，这里的名字不要手动输入，先点击Listing按钮，插件会在后台执行protoc命令，将你指定的proto根目录下的所有proto文件编译为pb文件，存放在一个临时目录下，一旦编译成功，点下拉框，就可以看到方法的列表，直接选中你要测试的就可以。<br>
<b>Deadline</b>：超时时间，这个可以适当调整长一些，默认只有1秒，一旦有gRPC请求处理超过一秒的，就会被强行关闭连接，导致请求报错，所以设置长一点对性能测试没有影响，可以避免一些异常。<br>
<b>Request</b>：以JSON格式写传递的参数。

需要注意的是，因为gRPC的特殊性，脚本写完了，如果交给别人，或者传到服务器上去跑，还是会失败的，因为必须要有proto文件来编译成pb文件。你在本地跑成功，是因为本地生成了临时目录存放pb文件，插件可以读取到，而临时目录一旦删除后（程序结束就会自动删除），就需要重新点一下listing按钮。所以放到服务器上或者交给别人运行，也需要把proto文件传过去才能正常运行，否则一定会报找不到pb文件的。<br>
