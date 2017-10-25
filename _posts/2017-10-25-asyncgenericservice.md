---
layout:     post
title:      "gRPC 之 AsyncGenericService 用法小结"
subtitle:   " \"AsyncGenericService \""
date:       2017-10-25
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - gRPC
    - C++
---

> AsyncGenericService 可以忽略特定的服务，接受任意的客户端请求，并做相应处理发送对应的响应。当有特定服务监听请求时，客户端的请求会被送到指定的服务，如果客户端的请求没有特定的服务监听，则均会被送到 AsyncGenericService 。

### 异步服务的常规用法

```C++
class CallData
{
public:
    CallData()
        : _status(CREATE)
    {
    }
    virtual void Proceed() = 0;
protected:
    enum CallStatus { CREATE, PROCESS, READ, WRITE, FINISH };
    CallStatus _status;
};
```
声明处理请求的基类，设置状态阶段码，后续通过状态码来执行不同阶段的操作。



```C++
class GreeterCallData : public CallData
{
public:
    GreeterCallData(std::shared_ptr<Greeter::AsyncService> service,
    				std::shared_ptr<ServerCompletionQueue> cq)
        : _service(service), _cq(cq), responder_(&ctx_)
    {
        Proceed();
    }

    void Proceed() override
    {
        if (_status == CREATE)
        {
            _status = PROCESS;
            _service->RequestSayHello(&ctx_, &request_, &responder_, _cq.get(), _cq.get(), this);
        }
        else if (_status == PROCESS)
        {
            new GreeterCallData(_service, _cq);

            std::string prefix("Hello ");
            reply_.set_message(prefix + request_.name());

            _status = FINISH;
            responder_.Finish(reply_, Status::OK, this);
        }
        else
        {
            GPR_ASSERT(_status == FINISH);
            delete this;
        }
    }
private:
    ServerContext ctx_;
    HelloRequest request_;
    HelloReply reply_;

    ServerAsyncResponseWriter<HelloReply> responder_;

    std::shared_ptr<ServerCompletionQueue>  _cq;
    std::shared_ptr<Greeter::AsyncService>  _service;
};
...
void HandleRpcs()
{
  new GenericCallData(_genericService, _cq);
  new GreeterCallData(_greeterService, _cq);
  void* tag;
  bool ok;
  while (true)
  {
    GPR_ASSERT(_cq->Next(&tag, &ok));
    GPR_ASSERT(ok);
    static_cast<CallData*>(tag)->Proceed();
  }
}
```

**解释：**

1. GreeterCallData 用于处理客户端请求，整个处理过程是异步的，通过轮询 ServerCompletionQueue 的通知，不断的调用 `Proceed()` 方法，并使用枚举状态标识 _status 来控制完成不同阶段的操作。
2. GreeterCallData 创建时 _status 为 `CREATE` 状态，进入 `CREATE` 分支。通过调用 Greeter::AsyncService 的 `RequestSayHello()` 方法（这里的SayHello是通过 protobuf 声明的服务名），向 ServerCompletionQueue 注册一个监听事件，然后将 _status 改为 `PROCESS` 状态。这里会将 ServerContext、HelloRequest、ServerAsyncResponseWriter 类型指针作为参数传入，用于存放调用的请求和上下文以及关联一个用于发送相应的 Writer 。随后的两个参数一个为 CompletionQueue 类型指针，用于处理请求，另一个为 ServerCompletionQueue  类型指针，用于通知新的请求，这两个参数可以是同一个指针。最后一个参数为 `void*` 类型的 tag，当 ServerCompletionQueue  的 `Next`  方法接收到新通知时，会被传递出来，以便后续的操作。
3. 客户端调用服务时，进入 `PROCESS` 分支，首先新建一个 GreeterCallData 对象用于处理下一个请求。客户端的上下文和请求会存放到 `RequestSayHello()` 方法传入的地址中，这时就可以取出并做相应操作。处理完需要响应的消息后，调用ServerAsyncResponseWriter 的 `Finish()` 方法，注册结束事件，传入需要发送的响应和状态码，并将 _status 改为 `FINISH`。 
4. 当有可写通知时，进入 `FINISH` 分支。此时，响应和状态码已经发送出去，可以删除对象来释放空间，至此GreeterCallData 整个的生存周期结束。



### AsyncGenericService 用法

```C++
class GenericCallData : public CallData
{
public:
    GenericCallData(std::shared_ptr<grpc::AsyncGenericService> service, std::shared_ptr<ServerCompletionQueue> cq)
        : _service(service)
        , _cq(cq)
    {
        _rw = new grpc::GenericServerAsyncReaderWriter(&_ctx);
        Proceed();
    }

    void Proceed() override
    {
        if (_status == CREATE)
        {
            _status = READ;
            _service->RequestCall(&_ctx, _rw, _cq.get(), _cq.get(), this);
        }
        else if (_status == READ)
        {
            std::cout << "method: " << _ctx.method() << std::endl;
            new GenericCallData(_service, _cq);

            _rw->Read(&_request, this);
            _status = WRITE;
        }
        else if (_status == WRITE)
        {
            Request req;
            Response response;
            ParseFromByteBuffer(&_request, &req);

            std::cout << req.fill_username() << std::endl;

            response.set_username("123456");
            std::unique_ptr<grpc::ByteBuffer> reply = SerializeToByteBuffer(&response);

            _rw->WriteAndFinish(*reply, grpc::WriteOptions(), grpc::Status::OK, this);
            _status = FINISH;
        }
        else
        {
            GPR_ASSERT(_status == FINISH);
            delete this;
        }
    }

private:
    grpc::GenericServerContext _ctx;
    grpc::ByteBuffer _request;
    grpc::GenericServerAsyncReaderWriter *_rw;

    std::shared_ptr<grpc::AsyncGenericService> _service;
    std::shared_ptr<ServerCompletionQueue> _cq;
};
```

**解释：**

AsyncGenericService 的用法和特定的 AsyncService 用法逻辑相同，不同之处在于上下文和数据的类型。这里使用了 GenericCallData 类来处理通用请求。

1. AsyncGenericService 注册监听事件的方法是 `RequestCall()` ， 前两个参数分别为 GenericServerContext 指针的上下文，GenericServerAsyncReaderWriter 指针的读写器。这里就不能传入存储请求的 request  参数，而是需要通过后续的方法来读取请求。后面的三个参数和特定的 `RequestXXX()` 方法的类型含义均相同。
2. 客户端调用服务时，首先任然新建 GenericCallData 对象用于监听下一个请求。然后需要调用 GenericServerAsyncReaderWriter 的 `Read()` 方法注册读请求事件，这里才传入了存储请求的 _request 指针，并且它是 `grpc::ByteBuffer` 类型的指针。
3. 当有读请求事件发生的通知时，反序列化 ByteBuffer 数据为相应的 Message 请求，就可以读取客户端发送的请求内容了。进行相应的操作后，调用 GenericServerAsyncReaderWriter 的 `WriteAndFinish()` 方法注册发送写和结束事件，传入需要发送的响应和状态码。
4. 接收到写和结束事件发生的通知，生命周期结束，释放自己。
5. 通过 GenericServerContext 的 `method()` 方法，可以获得该请求需要调用的服务名，服务端可以根据这个信息来将 ByteBuffer 反序列化为对应服务的请求。
6. 这里使用了多态的特性，当声明多个服务时，通过 tag 传入自身的指针，然后通过父类指针调用 Proceed 函数，进行相对应的处理。

### 客户端调用

客户端调用的方法和普通用法没有区别，可以同步也可以异步。

