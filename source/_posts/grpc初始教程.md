---
title: grpc初始教程
date: 2025-01-31 13:47:40
categories:
- go
---
[参考链接1](https://www.liwenzhou.com/posts/Go/gRPC/)对grpc有非常详细的教程，这里主要讲一些文章中没讲过的。
我们主要基于文章最后提供的链接分析一下其中的代码。
# 我们来看看.pb.go和_grpc.pb.go文件。
目前这是我们的pb文件。
```go
syntax = "proto3"; // 版本声明，使用Protocol Buffers v3版本

option go_package = "hello_server/pb";

package pb; // 包名


// 定义一个打招呼服务
service Greeter {
    // SayHello 方法
    rpc SayHello (HelloRequest) returns (HelloReply) {}
    // 服务端返回流式数据
    rpc LotsOfReplies(HelloRequest) returns (stream HelloReply);
    // 客户端发送流式数据
    rpc LotsOfGreetings(stream HelloRequest) returns (HelloReply);
    // 双向流式数据
    rpc BidiHello(stream HelloRequest) returns (stream HelloReply);
}

// 包含人名的一个请求消息
message HelloRequest {
    string name = 1;
}

// 包含问候语的响应消息
message HelloReply {
    string message = 1;
}
```
# pb.go
这个文件主要包含的是一些proto文件定义的类型和他的序列化方法。
```go
// 包含人名的一个请求消息
type HelloRequest struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Name string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
}
```
Name 后双引号内容的含义：
- bytes 指定字段的编码类型， 还可以是int32， int64， float，double， bool....
- 1表示的字段的编号
- name表示的字段名称
- opt表示是可选的，还有req（require），rep（repeated）

```go
func (x *HelloRequest) GetName() string {
	if x != nil {
		return x.Name
	}
	return ""
}
```
这里是Get我们自定义字段的方法。这个文件内其他的方法主要是序列化，反序列化，配合prorobuf使用。目前可以暂时不太关注。

# _ grpc.pb.go
## server
```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	// SayHello 方法
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
	// 服务端发送流式数据，客户端接收流式数据
	LotsOfReplies(*HelloRequest, Greeter_LotsOfRepliesServer) error
	// 客户端发送流式数据，服务端接收流式数据，并可以发送一次
	LotsOfGreetings(Greeter_LotsOfGreetingsServer) error
	// 双向流式数据
	BidiHello(Greeter_BidiHelloServer) error
	mustEmbedUnimplementedGreeterServer()
}
```
这是你需要实现的接口，然后流式API会新加入一些东西。
```go
// 这个接口是服务端发送流式数据
type Greeter_LotsOfRepliesServer interface {
	Send(*HelloReply) error
	grpc.ServerStream
}

type greeterLotsOfRepliesServer struct {
	grpc.ServerStream
}

func (x *greeterLotsOfRepliesServer) Send(m *HelloReply) error {
	return x.ServerStream.SendMsg(m)
}
```
```go
// 这里是服务端接收流式数据并可以发送一次，调用SendAndClose
type Greeter_LotsOfGreetingsServer interface {
	SendAndClose(*HelloReply) error
	Recv() (*HelloRequest, error)
	grpc.ServerStream
}

type greeterLotsOfGreetingsServer struct {
	grpc.ServerStream
}

func (x *greeterLotsOfGreetingsServer) SendAndClose(m *HelloReply) error {
	return x.ServerStream.SendMsg(m)
}

func (x *greeterLotsOfGreetingsServer) Recv() (*HelloRequest, error) {
	m := new(HelloRequest)
	if err := x.ServerStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
```
```go
// 这是双向流式数据，双方都可以发送流式数据
type Greeter_BidiHelloServer interface {
	Send(*HelloReply) error
	Recv() (*HelloRequest, error)
	grpc.ServerStream
}

type greeterBidiHelloServer struct {
	grpc.ServerStream
}

func (x *greeterBidiHelloServer) Send(m *HelloReply) error {
	return x.ServerStream.SendMsg(m)
}

func (x *greeterBidiHelloServer) Recv() (*HelloRequest, error) {
	m := new(HelloRequest)
	if err := x.ServerStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
```


## client
首先是client的接口和实现。
```go
// GreeterClient is the client API for Greeter service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://pkg.go.dev/google.golang.org/grpc/?tab=doc#ClientConn.NewStream.
type GreeterClient interface {
	// SayHello 方法
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
	// 服务端返回流式数据
	LotsOfReplies(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (Greeter_LotsOfRepliesClient, error)
	// 客户端发送流式数据
	LotsOfGreetings(ctx context.Context, opts ...grpc.CallOption) (Greeter_LotsOfGreetingsClient, error)
	// 双向流式数据
	BidiHello(ctx context.Context, opts ...grpc.CallOption) (Greeter_BidiHelloClient, error)
}

type greeterClient struct {
	cc grpc.ClientConnInterface
}

func NewGreeterClient(cc grpc.ClientConnInterface) GreeterClient {
	return &greeterClient{cc}
}
```
这里可以通过调用`NewGreeterClient`创建一个client。
```go
conn, err := grpc.Dial(*addr, grpc.WithInsecure())
if err != nil {
	log.Fatalf("did not connect: %v", err)
}
defer conn.Close()
c := pb.NewGreeterClient(conn)
```
`GreeterClient` 接口中也有很多结构体。接下来我们将分析分析其中的结构体。
### Greeter_LotsOfRepliesClient & LotsOfReplies
服务端流式数据, 同时还有对应方法的实现。
```go
type Greeter_LotsOfRepliesClient interface {
	Recv() (*HelloReply, error)
	grpc.ClientStream
}

type greeterLotsOfRepliesClient struct {
	grpc.ClientStream
}

func (x *greeterLotsOfRepliesClient) Recv() (*HelloReply, error) {
	m := new(HelloReply)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
func (c *greeterClient) LotsOfGreetings(ctx context.Context, opts ...grpc.CallOption) (Greeter_LotsOfGreetingsClient, error) {
	stream, err := c.cc.NewStream(ctx, &Greeter_ServiceDesc.Streams[1], "/pb.Greeter/LotsOfGreetings", opts...)
	if err != nil {
		return nil, err
	}
	x := &greeterLotsOfGreetingsClient{stream}
	return x, nil
}
```
### LotsOfGreetings & Greeter_LotsOfGreetingsClient
```go
type Greeter_LotsOfGreetingsClient interface {
	Send(*HelloRequest) error
	CloseAndRecv() (*HelloReply, error)
	grpc.ClientStream
}

type greeterLotsOfGreetingsClient struct {
	grpc.ClientStream
}

func (x *greeterLotsOfGreetingsClient) Send(m *HelloRequest) error {
	return x.ClientStream.SendMsg(m)
}

func (x *greeterLotsOfGreetingsClient) CloseAndRecv() (*HelloReply, error) {
	if err := x.ClientStream.CloseSend(); err != nil {
		return nil, err
	}
	m := new(HelloReply)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}

func (c *greeterClient) LotsOfGreetings(ctx context.Context, opts ...grpc.CallOption) (Greeter_LotsOfGreetingsClient, error) {
	stream, err := c.cc.NewStream(ctx, &Greeter_ServiceDesc.Streams[1], "/pb.Greeter/LotsOfGreetings", opts...)
	if err != nil {
		return nil, err
	}
	x := &greeterLotsOfGreetingsClient{stream}
	return x, nil
}
```
### BidiHello & Greeter_BidiHelloClient
```go
func (c *greeterClient) BidiHello(ctx context.Context, opts ...grpc.CallOption) (Greeter_BidiHelloClient, error) {
	stream, err := c.cc.NewStream(ctx, &Greeter_ServiceDesc.Streams[2], "/pb.Greeter/BidiHello", opts...)
	if err != nil {
		return nil, err
	}
	x := &greeterBidiHelloClient{stream}
	return x, nil
}

type Greeter_BidiHelloClient interface {
	Send(*HelloRequest) error
	Recv() (*HelloReply, error)
	grpc.ClientStream
}

type greeterBidiHelloClient struct {
	grpc.ClientStream
}

func (x *greeterBidiHelloClient) Send(m *HelloRequest) error {
	return x.ClientStream.SendMsg(m)
}

func (x *greeterBidiHelloClient) Recv() (*HelloReply, error) {
	m := new(HelloReply)
	if err := x.ClientStream.RecvMsg(m); err != nil {
		return nil, err
	}
	return m, nil
}
```
所以几乎就是开箱可用，如果你想继续深入了解grpc。
- 可以关注grpc本身提供的一些功能。比如grpc的的https验证，grpc的拦截器等等。
- 关注序列化和反序列化的逻辑，关注rpc本身的实现。



# 参考教程
1. https://www.liwenzhou.com/posts/Go/gRPC/
2. https://www.liwenzhou.com/posts/Go/Protobuf3-language-guide-zh/