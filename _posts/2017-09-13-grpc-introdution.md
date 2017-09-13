---
layout: post 
author: shalou
title:  "grpc学习" 
category: golang
tag: [golang,grpc]
---

grpc是google开源的一个高性能、跨语言的RPC框架，它是基于HTTP2.0协议，利用protocol buffers 3进行序列化。

* 高性能：基于protobuf3.0进行序列化，比利用Json性能快
* 跨语言：服务器端和客户端的实现可以是不同语言，grpc提供了工具让你可以自动生成客户端代码，以及部分服务器端代码

grpc中的远程调用主要分为四种

* 单项RPC：也叫做简单RPC，客户端发送一个请求，客户端则回应单个响应，然后关闭
* 服务器端流式RPC：客户端发送一个请求，服务器端得到一个stream，服务器端通过这个stream不停的发送数据给客户端，直到关闭，客户端可以用stream读取数据
* 客户端流式RPC：客户端通过一个stream发送数据给服务器端，服务器端接收到客户端所有请求后，回送一个应答给客户端
* 双向流式RPC：类似于websocket了，客户端和服务器端可以任意往stream写数据

<!-- more -->

## 1. 安装相应工具
我们利用go语言编程，要使用grpc，golang版本不低于v1.5

* 安装grpc

```golang
go get google.golang.org/grpc
```

* 安装protocol buffers v3

下载地址： [protobuf3](https://github.com/google/protobuf/releases)

* 安装go语言的proto插件

```golang
go get -u github.com/golang/protobuf/protoc-gen-go
```

protoc-gen-go会直接安装到$GOBIN目录下，你需要把该路径加到系统路径下，或者把该文件拷贝到/user/bin目录下

```golang
export PATH=$PATH:$GOPATH/bin
```



## 2. 利用proto定义RPC服务

创建一个文件user.protoc

```golang
syntax = "proto3";

message UserRequest {
    //用户ID
    int64 uid = 1;
}

message UserResponse {
    //用户名称
    string name   = 1;
    //用户年龄
    uint32 age    = 2;
    //用户性别
    uint32 gender = 3;
}

//List用户的条件
message UserCondition {
    //性别
    uint32 gender = 1;
}

message UserSummary {
    //描述
    string description = 1;
    //用户总数
    uint32 total       = 2;
}

message UserMessage {
    //谈话
    string talk = 1;
}

service User {
    //简单RPC
    rpc QueryUser(UserRequest) returns (UserResponse) {}

    //从服务端到客户端的stream rpc
    rpc ListUser(UserCondition) returns (stream UserResponse) {}

    //从客户端到服务端的stream rpc
    rpc SendUser(stream UserRequest) returns (UserSummary) {}

    // 双向stream rpc
    rpc Chat(stream UserMessage) returns (stream UserMessage) {}
}
```

在这里我们定义了一个User service，里面包含了四个RPC调用，分别是简单RPC，服务器端流式RPC，客户端流式RPC，双向流式RPC

下面可以利用protoc工具自动生成代码

```golang
protoc --go_out=plugins,grpc:. user.proto
```

会在当前目录下生成一个user.pb.go的文件，里面包含完整的客户端代码，以及服务器端需要实现的接口

## 3. 服务端实现

server.go

```golang
package main

import (
	"fmt"
	"io"
	"log"
	"net"

	pb "github.com/unionpay/user/user"

	context "golang.org/x/net/context"
	"google.golang.org/grpc"
)

const (
	address = "0.0.0.0:10023"
)

type UserService struct {
	users []*pb.UserResponse
}

func (svr *UserService) QueryUser(ctx context.Context, req *pb.UserRequest) (*pb.UserResponse, error) {
	fmt.Println("Query User")
	uid := req.GetUid()
	fmt.Println("User Id: ", uid)

	fmt.Printf("QueryUser end\n\n")

	return &pb.UserResponse{
		Name:   "Cloud",
		Age:    27,
		Gender: 0,
	}, nil
}

func (svr *UserService) ListUser(req *pb.UserCondition, stream pb.User_ListUserServer) error {
	fmt.Println("List User")
	gender := req.GetGender()
	fmt.Println("List User whose gender is: ", gender)

	for _, user := range svr.users {
		if user.Gender == gender {
			stream.Send(user)
		}
	}
	fmt.Printf("End List User\n\n")
	return nil
}

func (svr *UserService) SendUser(stream pb.User_SendUserServer) error {
	fmt.Println("SendUser")

	var count uint32 = 0
	for {
		user, err := stream.Recv()
		if err == io.EOF {
			fmt.Printf("server receive all users\n\n")
			//返回用户统计数据
			return stream.SendAndClose(&pb.UserSummary{
				Description: "total user",
				Total:       count,
			})
		}
		if err != nil {
			fmt.Println("server receive error: ", err)
		}

		fmt.Println("server receive user id ", user.GetUid())
		count++
	}
}

func (svr *UserService) Chat(stream pb.User_ChatServer) error {
	fmt.Println("Begin Chat")
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			fmt.Printf("chat done\n\n")
			return nil
		}

		if err != nil {
			fmt.Println("char error: ", err)
			return err
		}

		msg := in.GetTalk()

		fmt.Println("char message: ", msg)

		//回复
		stream.Send(&pb.UserMessage{
			Talk: "You Talk: " + msg,
		})
	}
}

func main() {
	listen, err := net.Listen("tcp", address)
	if err != nil {
		log.Fatal(err)
	}
    
   //定义user service
	svr := &UserService{
		users: []*pb.UserResponse{
			&pb.UserResponse{
				Name:   "Cloud",
				Age:    27,
				Gender: 0,
			},
			&pb.UserResponse{
				Name:   "Jim",
				Age:    26,
				Gender: 0,
			},
			&pb.UserResponse{
				Name:   "Tom",
				Age:    10,
				Gender: 0,
			},
			&pb.UserResponse{
				Name:   "Kate",
				Age:    20,
				Gender: 1,
			},
			&pb.UserResponse{
				Name:   "Ying",
				Age:    24,
				Gender: 1,
			},
		},
	}

	s := grpc.NewServer()

    //向grpc server注册服务
	pb.RegisterUserServer(s, svr)

	fmt.Println("Listening on the 0.0.0.0:10023")

    //启动server
	if err := s.Serve(listen); err != nil {
		log.Fatal(err)
	}
}
```

## 4.客户端调用

protoc自动为我们生成了客户端调用接口代码，我们只需要调用这些接口，就可以发送rpc请求

client.go

```golang
package main

import (
	"fmt"
	"io"
	"log"

	pb "github.com/unionpay/user/user"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
)

const (
	address = "172.17.138.101:10023"
)

func queryUser(client pb.UserClient) {
	fmt.Println("client QueryUser")
	user, err := client.QueryUser(context.Background(), &pb.UserRequest{Uid: 1})
	if err != nil {
		log.Fatal("QueryInfo error: ", err)
	}
	fmt.Println("user: ", user)
	fmt.Printf("client QueryUser End\n\n")
}

func listUser(client pb.UserClient) {
	fmt.Println("client ListUser")
	stream, err := client.ListUser(context.Background(), &pb.UserCondition{
		Gender: 1,
	})
	if err != nil {
		log.Fatal("ListUser stream error: ", err)
	}

	for {
		user, err := stream.Recv()
		if err == io.EOF {
			fmt.Printf("ListUser End\n\n")
			break
		}
		if err != nil {
			log.Fatal("ListUser error: ", err)
		}
		fmt.Println("user: ", user)
	}
}

func sendUser(client pb.UserClient) {
	fmt.Println("client sendUser")
	stream, err := client.SendUser(context.Background())
	if err != nil {
		log.Fatal("client SendUser create stream error: ", err)
	}

	queries := []*pb.UserRequest{
		&pb.UserRequest{
			Uid: 1,
		},
		&pb.UserRequest{
			Uid: 2,
		},
		&pb.UserRequest{
			Uid: 3,
		},
		&pb.UserRequest{
			Uid: 4,
		},
		&pb.UserRequest{
			Uid: 5,
		},
	}

	for _, query := range queries {
		err := stream.Send(query)
		if err != nil {
			log.Fatal("SendUser error: ", err)
		}
	}

	summary, err := stream.CloseAndRecv()
	if err != nil {
		log.Fatal("SendUser receive summary error: ", err)
	}

	fmt.Printf("Reveice summary: %s: %d\n", summary.GetDescription(), summary.GetTotal())
	fmt.Printf("SendUser End\n\n")
}

func chat(client pb.UserClient) {
	fmt.Println("Chat Start:")
	stream, err := client.Chat(context.Background())
	if err != nil {
		log.Fatal("client Chat get stream error: ", err)
	}

	chats := []*pb.UserMessage{
		&pb.UserMessage{
			Talk: "first talk",
		},
		&pb.UserMessage{
			Talk: "second talk",
		},
		&pb.UserMessage{
			Talk: "third talk",
		},
		&pb.UserMessage{
			Talk: "fourh talk",
		},
	}

	waitc := make(chan struct{})

	go func() {
		for {
			talk, err := stream.Recv()
			if err == io.EOF {
				close(waitc)
				return
			}

			if err != nil {
				log.Fatal("client Chat receive server chat error: ", err)
			}

			fmt.Println("server said: ", talk.GetTalk())
		}
	}()

	for _, chat := range chats {
		err := stream.Send(chat)
		if err != nil {
			log.Fatal("client Chat send chat error: ", err)
		}
	}

	//结束对话
	err = stream.CloseSend()
	if err != nil {
		log.Fatal("client Chat close chat error: ", err)
	}

	<-waitc
	fmt.Printf("client Chat end\n\n")
}

func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatal("dail error:", err)
	}
	defer conn.Close()

	c := pb.NewUserClient(conn)

	queryUser(c)
	listUser(c)
	sendUser(c)
	chat(c)

}
```
