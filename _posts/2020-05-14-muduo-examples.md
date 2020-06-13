---
title: "muduo"
published: true
---

## Intro ##

    构建一个分布式服务器，可借鉴的muduo代码

**URL**

    https://github.com/chenshuo/muduo
    https://github.com/chenshuo/muduo-tutorial

**AsyncLogging**

    muduo-tutorial/src/echo.cc

**ProtobufCodec**

    examples/protobuf/codec/
    
**ThreadPool, setHighWaterMarkCallback, setWriteCompleteCallback**

    examples/sudoku/server_prod.cc
    
**PingPong**

    examples/pingpong/server.cc
    examples/pingpong/client.cc //EventLoopThreadPool, throughput
    
    examples/ace/logging/server.cc
    
    examples/sudoku/batch.cc //QPS
    
**CountDownLatch**

    examples/memcached/client/bench.cc //EventLoopThreadPool, QPS, pingpong
    examples/protobuf/rpcbench/client.cc
    examples/wordcount/hasher.cc

**outstandings_**

    examples/protobuf/rpcbalancer/balancer.cc
    examples/protobuf/rpcbalancer/balancer_raw.cc
    muduo/net/protorpc/RpcChannel.cc
         CallMethod
         onRpcMessage

// wire format
//
// Field     Length  Content
//
// size      4-byte  M+N+4
// tag       M-byte  could be "PB", "MP", etc.
// payload   N-byte
// checksum  4-byte  adler32 of tag+payload

message PbMessage
{
  required MessageType type = 1;
  required fixed64 id = 2;

  optional string typeName = 3;
  optional bytes request = 4;

  optional bytes response = 5;

  optional ErrorCode error = 6;
}

