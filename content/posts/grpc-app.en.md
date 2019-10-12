---
title: "gRPC App"
date: 2019-10-07T18:45:40+11:00
draft: true
tags: ["go", "gRPC", "application"]
description: An introductary gRPRC server application
---

The previous post discussed the motivations and benefits of using gRPC. This post will focus on getting started building gRPC applications in [go](https://golang.org/). Most of this post will run parallel to the go documentation on the [gRPC examples website](https://github.com/grpc/grpc-go/tree/master/examples). This post will deviate from those docs particularly in handling bidirectional requests in a concurrent non-blocking fashion.

## Before you start

Make sure go is installed on your computer. You will also need to install gRPC, go protobuf generator. To get setup you can follow install documentation [here](https://grpc.io/docs/quickstart/go/).

---

## Project scaffold

This project will be much simpler then the application we'll build in the next post; as such, we will keep the directory structure simple. We will add most of the source code to just two files. While this is a great way to start building software it is worth pointing out that this structure will not scale to larger projects.

Every project needs a mod file so create that

```bash
# replace my git repository with your own
go mod init github.com/schafer14/grpc-example
```

## The proto file

Let us start by creating a module that will hold our proto file and all the generated code associated with it. Create a file `requests/service.proto`. gRPC supports unary, server streaming, client streaming, and bidirectional streaming requests. We will create a single endpoint for each request type.

```protobuf
syntax="proto3";

message Request {
    string body = 1;
    string from = 2;
}

message Empty{}

service RequestService {
    rpc GetRequest(Empty) returns (Request) {}
    rpc ServerStreamRequests(Empty) returns (stream Request) {}
    rpc ClientStreamRequests(stream Request) returns (Request) {}
    rpc BidirectionalRequests(stream Request) returns (stream Request) {}
}
```

First two types are defined for use in our service, a request type and an empty type. The request type has a couple fields and the empty type is empty. It is worth noting that even though all fields are strings in this example protobufs support a rich [list of types](https://developers.google.com/protocol-buffers/docs/proto3) and support utilities to make composite types.

Next a service is created. This contains a list of functions that our service will implement. We use one function for each of the four types of functions gRPC supports. Notice that streams are denoted by the `stream` keyword. All other requests are assumed to be single messages.

We are now ready to generate the supporting code for our service.

```bash
protoc --go_out=plugins=grpc:. requests/service.proto
```

This creates a second file in our requests module. This file defines an interface that our service must implement, once that service is implemented the generated code contains functions for both servers and clients to interact with each other.

## Building a server

Building a server that serves this service is pretty straight forward now. We need to implement the interface generated in the previous step, and then define some configuration to run the server.

A good starting point is implementing the generated interface. This interface has a single function for each function in our proto file. We will go through them one by one. This code will be in a new module so create a file `server/main.go` in the root directory.

Implement a struct that will implement the generated interface.

```go
// server/main.go
package main

import (
    // replace with the git repository you used in the go mod init command
    pb "github.com/schafer14/grpc-example/requests"
    // ...
)

type requestService struct {
    // this is a good place for dependencies such as databases and loggers
}

func newService() requestService {
    return requestService{}
}
```

This begins to describe the struct `requestService` that will implement all the functions we need to run our service.

### GetRequests (Unary)

```go
// GetRequest returns a single request object
func (requestService) GetRequest(ctx context.Context, req *pb.Empty) (*pb.Request, error) {
    return &pb.Request{From: "Server", Body: "I have a single message for you."}, nil
}
```

If you are not used to working with protobufs this snippet might look strange, but it is not complicated at all. Remember the file that `protoc` generated for us? It defined a new type for each of the messages in our service. All we are doing here is using those types. If you ever wonder what type you need referring to that file can be very helpful.

Since this function has a signature that takes a single empty object and returns a single request object without any streams the function is familiar.

### ServerStreamRequests (Server Streaming)

In this type of request the server will stream requests to the client. Either the server or the client can end the connection at any time.

```go
// ServerStreamRequests streams 100 request objects to the client
func (requestService) ServerStreamRequests(req *pb.Empty, stream pb.RequestService_ServerStreamRequestsServer) error {
    for i := 1; i <= 100; i++ {
        select {
        case <-stream.Context().Done():
            log.Printf("Connection closed by client\n")
            return nil
        default:
            msg := &pb.Request{From: "Streaming Server", Body: fmt.Sprintf("Message %v", i)}
            stream.SendMsg(msg)
            fmt.Printf("Sending %v\n", i)
            time.Sleep(250 * time.Millisecond)
        }
    }
    return nil
}
```

Examining the function signature we immediately notice a difference from the unary request. First, there is no context parameter, but we can still get the context with `stream.Context()`. If you are unfamiliar with context make sure to checkout [this video](https://www.youtube.com/watch?v=LSzR0VEraWw). The second change is that instead of returning the result there is a stream parameter that can be use to send messages to the client.

If our goal is to send 100 messages to the client and we simply use a for loop to send messages the server will continue to try to send messages even if the client has disconnected. Instead after each message we send we want to check if the client is still listening and if so send the message. This ensures are server will no waste valuable resource computing messages that will essentially be piped to `/dev/null`.

### ClientStreamRequests (Client Streaming)

In this function the client will send messages to the server. In our example the server will only listen for a maximum of five seconds. At the end of the listening period the server will send a response back to the client.

```go
// ClientStreamRequests listens to messages from a client and then at the end of the messages sends
// a response to notify of receipt.
func (requestService) ClientStreamRequests(stream pb.RequestService_ClientStreamRequestsServer) error {
    // Set a maximum of five seconds to listen to the client
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    // cancel context when the function ends regardless of what happens
    defer cancel()

    // We will need to listen to messages in a separate routine so create a channel
    // to syncronize the two routines
    inChan := make(chan pb.Request)
    // Create a separate routine that just listens for messages from the client
    go func(ch chan pb.Request) {
        for {
            message, err := stream.Recv()
            if err != nil {
                // cancel the context so the main loop will know to stop
                cancel()
                return
            }
            ch <- *message
        }
    }(inChan)

Loop:
    for {
        select {
        case <-ctx.Done():
            break Loop
        case <-stream.Context().Done():
            log.Println("Closed by client")
            break Loop
        case message := <-inChan:
            log.Println(&message)
        }
    }

    // Close connection
    log.Println("Final response")
    return stream.SendAndClose(&pb.Request{From: "Listening Server", Body: "Done"})
}
```

This is a little bit more complicated then previous functions because there is a need to do two things at once:

- Listen to messages from the client
- Listen for cancellations or deadlines from context

In order this we set up a separate go routine to monitor messages from the client. Apart from that complication, the signature and overall mechanics of the function are very similar to the previous example.

### BidirectionalRequests (Bidirectional Streaming)

As you may have guessed bidirectional streaming is just a combination of client streaming and server streaming. The following code may look imposing, but if you understand the previous two functions there is nothing new. We need to create two go routines. One for generating messages and one for receiving messages. This also means the select statement will have an extra branch.

```go
func (requestService) BidirectionalRequests(stream pb.RequestService_BidirectionalRequestsServer) error {
    // Set the context for cancellation
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Create a channel to read from client
    inChan := make(chan pb.Request)
    go func(input chan pb.Request) {
        for {
            in, err := stream.Recv()
            if err != nil {
                cancel()
                return
            }
            input <- *in
        }
    }(inChan)

    // Create a channel to generate messages
    outChan := make(chan pb.Request)
    go func(out chan pb.Request) {
        for i := 0; i < 100; i++ {
            select {
            case <-ctx.Done():
                return
            default:
                out <- pb.Request{From: "Bidirectional Server", Body: fmt.Sprintf("Message %v", i)}
                time.Sleep(time.Millisecond * 150)
            }
        }
    }(outChan)

    // react to whatever is ready to go.
    for {
        select {
        case <-ctx.Done():
            log.Println("Closed by server")
            return nil
        case <-stream.Context().Done():
            log.Println("Closed by client")
            return nil
        case message := <-outChan:
            stream.Send(&message)
        case msg := <-inChan:
            log.Println(&msg)
            stream.Send(&pb.Request{From: "Server", Body: "I got your message!"})
        }
    }
}
```

### The main function

With the hard part out of the way we can focus on initiating the gRPC application. This is pretty standard so I will not provide as many annotations.

```go
func main() {
    host := flag.String("host", ":8080", "The server host")

    flag.Parse()

    lis, err := net.Listen("tcp", *host)

    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    grpcServer := grpc.NewServer()

    pb.RegisterRequestServiceServer(grpcServer, newService())

    log.Printf("Listening on %v\n", *host)
    log.Fatal(grpcServer.Serve(lis))
}
```

---

Hopefully this has been a clear demonstration of how to use gRPC. If you are looking for the source code or a client implementation it can be found on [github](https://github.com/schafer14/grpc-example). In the next post we will give an example of how these techniques could be used in a real application.

