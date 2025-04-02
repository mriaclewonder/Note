# Chat



## 客户端传包

![image-20250401183639009](D:/Code/TYPORA/Note/Chat.assets/image-20250401183639009.png)

## 什么是大端，什么是小端

低字节：数值的**低位部分**（Least Significant Byte, LSB），即对数值大小影响最小的字节。

以	0x12345678 为例    

​	0x12 是高字节

​	0x78 是低字节

低地址：内存中某个存储单元（如变量、数组）的**起始位置**，或地址较小的位置。

#### **小端模式（Little-Endian）**

- **规则**：**低字节**存储在**低地址**，**高字节**存储在**高地址**。

#### **大端模式（Big-Endian）**

- **规则**：**高字节**存储在**低地址**，**低字节**存储在**高地址**。

## 为什么喜欢用队列去处理异步？

因为队列可以保证异步的有序性

## **rpc和grpc**

**RPC（Remote Procedure Call，远程过程调用）是一种通信协议**，允许程序像调用本地函数一样调用远程服务器上的函数或方法。它抽象了网络通信的复杂性，使得开发者无需直接处理数据传输、序列化或网络协议细节，只需关注业务逻辑。RPC的核心思想是让跨进程或跨网络的函数调用对开发者透明，常见实现方式包括基于TCP、HTTP等协议，适用于微服务架构、分布式系统等场景，例如服务A调用服务B提供的接口。典型的RPC框架（如Dubbo、Thrift）会封装通信过程，提供序列化、服务发现、负载均衡等功能，但不同框架在性能、跨语言支持和协议设计上各有侧重。

**gRPC是Google基于RPC理念开发的高性能开源框架**，默认采用HTTP/2协议实现双向流式通信，并依赖Protocol Buffers（Protobuf）作为接口定义语言（IDL）和序列化工具。与传统RPC框架相比，gRPC通过HTTP/2的多路复用、头部压缩等特性显著提升传输效率，同时Protobuf的二进制编码比JSON/XML更紧凑高效，适合高并发、低延迟场景。此外，gRPC支持跨语言开发（如C++、Java、Python、Go等），通过预编译Protobuf文件自动生成客户端和服务端代码，简化了多语言协作的复杂性。其核心能力还包括双向流式调用（如客户端流、服务端流、双向流），适用于实时通信、IoT设备控制等场景，已成为云原生和微服务领域的主流通信方案之一。

gRPC 的完整流程可以概括为以下几个阶段，涵盖从接口定义到实际通信的全过程：

------

### **1. 定义服务接口（Proto 文件）**

- **作用**：使用 Protocol Buffers（Protobuf）编写 `.proto` 文件，明确服务的方法、请求和响应消息格式。

  ```protobuf
  syntax = "proto3";
  service MyService {
    rpc GetData (Request) returns (Response) {}  // 定义 RPC 方法
  }
  message Request { string query = 1; }          // 请求结构
  message Response { string result = 1; }       // 响应结构
  ```

- **意义**：该文件是跨语言协作的契约，也是生成代码的输入。

------

### **2. 生成桩代码（Stub Code）**

- **工具**：通过 `protoc` 编译器配合 `grpc_cpp_plugin`（或其他语言插件）生成代码。

  生成文件

  - **消息类**（如 `Request.pb.h`）：序列化请求/响应消息的代码。
  - **服务类**（如 `MyService.grpc.pb.h`）：包含服务端基类（`Service`）和客户端桩（`Stub`）。

  命令示例

  ```bash
  protoc --grpc_out=. --cpp_out=. --plugin=protoc-gen-grpc=grpc_cpp_plugin my_service.proto
  ```

------

### **3. 实现服务端**

- 步骤

  1. **继承生成的服务基类**：重写定义的 RPC 方法（如 `GetData`），填充业务逻辑。
  2. **启动服务**：绑定端口，注册服务实例，监听客户端请求。

- 代码要点

  ```cpp
  class MyServiceImpl final : public MyService::Service {
    Status GetData(ServerContext* context, const Request* req, Response* res) override {
      res->set_result("Processed: " + req->query());  // 处理请求
      return Status::OK;
    }
  };
  // 启动服务器
  ServerBuilder builder;
  builder.AddListeningPort("0.0.0.0:50051", grpc::InsecureServerCredentials());
  builder.RegisterService(&service);
  std::unique_ptr<Server> server(builder.BuildAndStart());
  ```

------

### **4. 实现客户端**

1. **创建通道（Channel）**：指定服务端地址（如 `localhost:50051`）。
2. **调用桩方法**：通过生成的 `Stub` 对象发起 RPC 调用，处理响应或错误。

- 代码要点

  ```cpp
  auto channel = grpc::CreateChannel("localhost:50051", grpc::InsecureChannelCredentials());
  MyService::Stub stub(channel);
  Request request;
  request.set_query("test");
  Response response;
  ClientContext context;
  Status status = stub.GetData(&context, request, &response);
  if (status.ok()) {
    std::cout << response.result();  // 输出结果
  }
  ```

------

### **5. 通信过程（基于 HTTP/2）**

1. **建立连接**：客户端与服务端通过 TCP 建立 HTTP/2 连接。

2. **多路复用**：单个连接上并行处理多个请求/响应（减少连接开销）。

3. 传输数据

   - 客户端将 `Request` 序列化为二进制 Protobuf 数据，通过 HTTP/2 流发送。
   - 服务端反序列化数据，执行对应方法，返回序列化后的 `Response`。

4. 流式支持

   （可选）：

   - **单向流**：客户端或服务端分多次发送数据（如实时日志传输）。
   - **双向流**：双方同时通过流发送数据（如聊天应用）。

------

### **6. 关键流程总结**

1. **定义接口** → **生成代码** → **实现服务端逻辑** → **客户端调用**。
2. **底层依赖**：HTTP/2 提供高效传输，Protobuf 保证紧凑的序列化。
3. **核心优势**：跨语言、高性能、支持复杂通信模式（如流式）。

------

### **示例类比（简化版）**

想象客户端像“寄信”：

1. **写请求**（序列化）：将数据按 Protobuf 格式打包。
2. **找邮局**（创建通道）：连接到服务端地址。
3. **投递**（调用 Stub）：通过 HTTP/2 “快递”发送请求。
4. **处理信件**：服务端拆包、处理、返回结果。
5. **收件**（反序列化）：客户端解析响应数据。