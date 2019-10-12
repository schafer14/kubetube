---
title: "Grpc Intro"
date: 2019-10-01T16:34:36+11:00
draft: true
tags: ["go", "gRPC", "intro"]
---

[gRPC](https://grpc.io/) is an exciting new transport layer technology. Development of enabling technologies has led to developers rethinking on how to build software. gRPC is one of these enabling technologies that have a significant impact on designing software and networks using more robust and concise patterns. gRPC stands out to me as a cornerstone for building applications that work and integrate better with users and other applications.
In this series, we are going to take a look at what gRPC is, how it’s different from similar technologies and build an application that utilizes gRPC to improve on existing applications. Here’s an outline of what is ahead:

- An introduction to gRPC - discussion on alternate technologies and supporting technologies for gRPC
- Building a simple application - this expands upon the documentation give on the gRPC website with a focus on how to use Go features such as context with timeouts and channels with gRPC.
- Intermezzo: An optional lesson on building a chess engine in Go.
- Building a chess application - we look at shortcomings of chess.com and Lichess and build a chess application with a little twist.
- Deploying gRPC - addresses the pros and cons of deploying gRPC with different services, we deploy our application using Knative.

---

## Surrounding Technologies


No technology lives in a vacuum. gRPC operates on top of [http2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) using [protobufs](https://developers.google.com/protocol-buffers/) as a wire format. A strong abstraction of the http2 protocol makes it possible to use gRPC with little or no knowledge about how http2 works; however, a knowledge of the protobuf service syntax is essential for working with gRPC. Fortunately, protobufs is easy to use, and the benefits that it provides are substantial. 

Protobufs are a mechanism for serializing data. The key component is a `.proto` file that defines some services and message types. Here is an example from the developer guides of a proto file:

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

Using this definition, we can serialize a search request for transportation. If this is hard to understand think about using JSON.serialize to turn a JavaScript object into JSON. This protobuf is a function to turn a search request object into a text blob. 

While JSON has been a popular mechanism for serializing data, protobufs offers a few advantages:

- Type safety - to serialize correctly, an object must validate against the protobuf definition
- Code generation - source code can be automatically generated because the type (inputs and outputs) of the service is well-defined.
- Smaller data size - using the message definition, protobufs, can serialize objects intelligently.  

There are some trade-offs when adopting protobufs. Here are some of the downsides of protobufs.

- Both the server and client need to know where the `.proto` file is.
- The `.proto` file needs to be kept in sync across all applications using it (although there are migration [policies](https://developers.google.com/protocol-buffers/docs/proto3#updating) that can mitigate this issue).
- Data is difficult to read once serialized. 

Additionally, there is a quirk of protobufs that can lead to some confusion. Empty values get de-serialized to default values; this makes perfect sense if you think about trying to provide a message format that can be used by both strongly and weakly typed languages, but it can be confusing in particular with boolean values. A good trick to remember is always defaulting to false for boolean fields. 

## gRPC

gRPC is essentially the combination of http2 and protobufs. It supports a handful of excellent features, but the real benefits to my eyes are streaming messages. gRPC supports request-response communication, but it also supports client streaming, server streaming, and bi-directional streaming communication. These properties facilitate the development of a whole bunch of new applications.

While streaming data is a nice feature, it is not a new idea, and there are a few alternative ways to do so. Let’s have a look.

### Differences with WebSockets

Websockets have been my go-to solution for real-time applications in the past. The big difference I see between WebSockets and gRPC is WebSockets are untyped. Being untyped means there is no control over the information flowing between the services; this can lead to systems that are very easy to build but difficult to maintain. Often this leads to poor documentation and systems that are difficult to understand.

Additionally, gRPC uses protobufs as it’s interchange format meaning that no additional work is done getting messages into reasonably small formats. In my experience, most WebSocket services use JSON and have poor or no documentation. In these cases, gRPC almost automatically increases the type-safety and performance with no additional developer effort. 

### Differences with message brokers

An alternative way to do data streaming is to use a message system like [RabbitMQ](https://www.rabbitmq.com/), [NATS](https://nats.io/), or one of many others. While these are certainly not a like for like replacement of gRPC, they do bring up some interesting points about gRPC. First, gRPC, in most cases, is a library not a separate piece of software; this gives tighter coupling with your application which can be good or bad, but there are new frameworks such as [go kit](https://gokit.io/) and [go micro](https://micro.mu/docs/go-micro.html) that are finding innovative ways of decoupling gRPC from application logic. When using gRPC, a client requires knowledge of the host server; in contrast, when using service discovery systems can communicate without explicit knowledge of each other. Service discovery can mitigate this issue, but it is something to consider when using gRPC. 

---

This post was a summary of the concepts and considerations associated with gRPC. Understanding it gives theoretical knowledge of gRPC and it’s underlying technologies. This post lays the groundwork for building applications using gRPC, which we start doing in the next post.