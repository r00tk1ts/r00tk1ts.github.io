---
title: Chromium-sandbox-SharedMemIPC-analysis
date: 2018-05-20 08:31:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第八篇，主要分析了windows平台下，Chromium sandbox IPC通信中使用的共享内存IPC机制，其中分析了IPC buffer的设计以及C/S两端的使用流程。阅读本篇前，请先阅读第七篇。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-SharedMemIPC-analysis

`CrossCall`的server和client只组成了IPC通信的一部分，尽管整体的通信模型已经有了了解，但代码拼图还未完成，上一节留下的疑问是Server端的`Dispatcher`是如何驱动的，又有谁来驱动。经一番搜索，在`SharedMemIPC`相关的代码中找到了答案。

`SharedMemIPC`是一个基于共享内存的IPC机制，它是IPC的一种具体实现的封装。也可以通过其他机制来实现IPC，比如命名管道等。

## Server

且看sharedmem_ipc_server.h对server的描述：

```cpp
// IPC transport implementation that uses shared memory.
// This is the server side
//
// The server side has knowledge about the layout of the shared memory
// and the state transitions. Both are explained in sharedmem_ipc_client.h
// server对共享内存的布局是知情的。
// 
// As opposed to SharedMemIPClient, the Server object should be one for the
// entire lifetime of the target process. The server is in charge of creating
// the events (ping, pong) both for the client and for the target that are used
// to signal the IPC and also in charge of setting the initial state of the
// channels.
// 与Client不同的是，对整个target进程生命周期来说，server object只能有一个
// 它负责为client和用于signal IPC的target创建事件，也负责设置channel（信道）的初始化状态
// 
// When an IPC is ready, the server relies on being called by on the
// ThreadPingEventReady callback. The IPC server then retrieves the buffer,
// marshals it into a CrossCallParam object and calls the Dispatcher, who is in
// charge of fulfilling the IPC request.
// IPC到来时，server在ThreadPingEventReady回调函数上回复。
// IPC server解包buffer，置入CrossCallParam对象并调用Dispatcher来处理IPC请求。
```

Server处理的后半段机制已经在CrossCall的server端看到了。感兴趣的是这个`ThreadPingEventReady`。

### `SharedMemIPCServer`

先看一下类定义头：

```cpp
// the shared memory implementation of the IPC server. There should be one
// of these objects per target (IPC client) process
class SharedMemIPCServer {
 public:
  // Creates the IPC server.
  // target_process: handle to the target process. It must be suspended. It is
  // unfortunate to receive a raw handle (and store it inside this object) as
  // that dilutes ownership of the process, but in practice a SharedMemIPCServer
  // is owned by TargetProcess, which calls this method, and owns the handle, so
  // everything is safe. If that changes, we should break this dependency and
  // duplicate the handle instead.
  // target_process_id: process id of the target process.
  // thread_provider: a thread provider object.
  // dispatcher: an object that can service IPC calls.
  // 构造器很关键，数据结构的关联就在此处理，描述中得知一个关键的线索是SharedMemIPCServer由TargetProcess所持有
  SharedMemIPCServer(HANDLE target_process,				//关联target进程句柄和id
                     DWORD target_process_id,
                     ThreadProvider* thread_provider,	//关联线程池
                     Dispatcher* dispatcher);			//关联分发器

  ~SharedMemIPCServer();

  // Initializes the server structures, shared memory structures and
  // creates the kernels events used to signal the IPC.
  // 看来还是有日常操作Init，用于初始化共享内存的尺寸、信道分割
  bool Init(void* shared_mem, uint32_t shared_size, uint32_t channel_size);

 private:
  // Allow tests to be marked DISABLED_. Note that FLAKY_ and FAILS_ prefixes
  // do not work with sandbox tests.
  FRIEND_TEST_ALL_PREFIXES(IPCTest, SharedMemServerTests);
  // When an event fires (IPC request). A thread from the ThreadProvider
  // will call this function. The context parameter should be the same as
  // provided when ThreadProvider::RegisterWait was called.
  // IPC请求到来时，ThreadProvider产生的thread会先调用这个static函数
  // context就是ThreadProvider::RegisterWait调用时用到的context
  static void __stdcall ThreadPingEventReady(void* context, unsigned char);

  // Makes the client and server events. This function is called once
  // per channel.
  // C/S之间的来回event make方法，因为是共享内存的IPC，所以填充的一方signal ping事件，
  // 表示收端可以处理了，而收端处理完signal pong事件，表示填充完结果请发端get
  // ping-pong的名称很有灵性啊
  bool MakeEvents(base::win::ScopedHandle* server_ping,
                  base::win::ScopedHandle* server_pong,
                  HANDLE* client_ping,
                  HANDLE* client_pong);

  // A copy this structure is maintained per channel.
  // Note that a lot of the fields are just the same of what we have in the IPC
  // object itself. It is better to have the copies since we can dispatch in the
  // static method without worrying about converting back to a member function
  // call or about threading issues.
  // 定义了一个内部结构体，提取了一些IPC对象本体的成员，用于在static方法中dispatch
  struct ServerControl {
    ServerControl();
    ~ServerControl();

    // This channel server ping event.
    base::win::ScopedHandle ping_event;
    // This channel server pong event.
    base::win::ScopedHandle pong_event;
    // The size of this channel.
    uint32_t channel_size;
    // The pointer to the actual channel data.
    char* channel_buffer;
    // The pointer to the base of the shared memory.
    char* shared_base;
    // A pointer to this channel's client-side control structure this structure
    // lives in the shared memory.
    ChannelControl* channel;
    // the IPC dispatcher associated with this channel.
    Dispatcher* dispatcher;
    // The target process information associated with this channel.
    ClientInfo target_info;
  };

  // Looks for the appropriate handler for this IPC and invokes it.
  // 这个是对IPC handler的查找与调用static方法
  static bool InvokeCallback(const ServerControl* service_context,
                             void* ipc_buffer,
                             CrossCallReturn* call_result);

  // Points to the shared memory channel control which lives at
  // the start of the shared section.
  IPCControl* client_control_;	//共享内存的起始结构，控制了channel的分割

  // Keeps track of the server side objects that are used to answer an IPC.
  // 这里把ServerControl对象的指针做成链，在响应IPC时会用到
  std::list<std::unique_ptr<ServerControl>> server_contexts_;

  // The thread provider provides the threads that call back into this object
  // when the IPC events fire.
  // 当IPC事件到来时，thread_provider_就是那个负责发起线程来调用callback的线程池
  ThreadProvider* thread_provider_;

  // The IPC object is associated with a target process.
  // IPC对象关联的target进程句柄
  HANDLE target_process_;

  // The target process id associated with the IPC object.
  // IPC对象关联的target进程id
  DWORD target_process_id_;

  // The dispatcher handles 'ready' IPC calls.
  // Dispatcher就是IPC call的分发器，分发到具体的callback
  Dispatcher* call_dispatcher_;

  DISALLOW_COPY_AND_ASSIGN(SharedMemIPCServer);
};
```

构造器/析构器：

```cpp
// 显然thread_provider和dispatcher都不归该对象管理，SharedMemIPCServer只是他们的操纵者
SharedMemIPCServer::SharedMemIPCServer(HANDLE target_process,
                                       DWORD target_process_id,
                                       ThreadProvider* thread_provider,
                                       Dispatcher* dispatcher)
    : client_control_(nullptr),
      thread_provider_(thread_provider),
      target_process_(target_process),
      target_process_id_(target_process_id),
      call_dispatcher_(dispatcher) {
  // We create a initially owned mutex. If the server dies unexpectedly,
  // the thread that owns it will fail to release the lock and windows will
  // report to the target (when it tries to acquire it) that the wait was
  // abandoned. Note: We purposely leak the local handle because we want it to
  // be closed by Windows itself so it is properly marked as abandoned if the
  // server dies.
  if (!g_alive_mutex) {
    HANDLE mutex = ::CreateMutexW(nullptr, true, nullptr);
    // 教科书般的竞态处理
    if (::InterlockedCompareExchangePointer(&g_alive_mutex, mutex, nullptr)) {
      // We lost the race to create the mutex.
      ::CloseHandle(mutex);
    }
  }
}

SharedMemIPCServer::~SharedMemIPCServer() {
  // Free the wait handles associated with the thread pool.
  // 暂时先不关心thread_provider的实现机制，了解他的角色功用就行了
  if (!thread_provider_->UnRegisterWaits(this)) {
    // Better to leak than to crash.
    return;
  }
  server_contexts_.clear();

  if (client_control_)
    ::UnmapViewOfFile(client_control_);	// 明显共享内存是client控制的，它是IPCControl起始的结构
}
```

明显析构和构造对不上，这是chrome的固有套路，构造要接Init:

```cpp
bool SharedMemIPCServer::Init(void* shared_mem,
                              uint32_t shared_size,
                              uint32_t channel_size) {
  // The shared memory needs to be at least as big as a channel.
  // 共享内存大小至少得容纳一个channel
  if (shared_size < channel_size) {
    return false;
  }
  // The channel size should be aligned.
  // 信道尺寸必须得是按32字节对齐
  if (0 != (channel_size % 32)) {
    return false;
  }

  // Calculate how many channels we can fit in the shared memory.
  // 计算出共享内存可以容纳多少个信道（每个信道一次跑一个IPC调用）
  // 这里的计算方式与IPCControl结构有关，channels是IPCControl的最后一个flexible ChannelControl数组成员
  // 扣除IPCControl的其他成员，剩余的尺寸都是承载ChannelControl的
  // 关于IPCControl和ChannelControl，它们的结构在下面分析。
  // 简单来说就是IPCControl控制了有多少个ChannelControl，而每个ChannelControl又控制了真实的channel buffer的偏移位置
  // 这和crosscall的设计很像
  shared_size -= offsetof(IPCControl, channels);
  size_t channel_count = shared_size / (sizeof(ChannelControl) + channel_size);

  // If we cannot fit even one channel we bail out.
  // 一个信道都放不下，are you kidding me?
  if (0 == channel_count) {
    return false;
  }
  //算出第一个真实信道的起始位置
  // Calculate the start of the first channel.
  size_t base_start =
      (sizeof(ChannelControl) * channel_count) + offsetof(IPCControl, channels);

  // 调整client_control_这个IPCControl指针指向了共享内存头部
  client_control_ = reinterpret_cast<IPCControl*>(shared_mem);
  client_control_->channels_count = 0;

  // This is the initialization that we do per-channel. Basically:
  // 1) make two events (ping & pong)
  // 2) create handles to the events for the client and the server.
  // 3) initialize the channel (client_context) with the state.
  // 4) initialize the server side of the channel (service_context).
  // 5) call the thread provider RegisterWait to register the ping events.
  // 每个channel都要做两个事件(ping & pong)
  // client对事件句柄的保存在client_context中，也就是shared_mem
  // server对事件句柄的保存在new出来的ServerControl中，每个ServerControl都丢入server_contexts_
  for (size_t ix = 0; ix != channel_count; ++ix) {
    ChannelControl* client_context = &client_control_->channels[ix];
    ServerControl* service_context = new ServerControl;
    server_contexts_.push_back(base::WrapUnique(service_context));//丢入的是智能指针

    if (!MakeEvents(&service_context->ping_event, &service_context->pong_event,
                    &client_context->ping_event, &client_context->pong_event)) {
      return false;
    }

    client_context->channel_base = base_start;
    client_context->state = kFreeChannel;	//ChannelState这个enum记录了channel的几个state

    // Note that some of these values are available as members of this object
    // but we put them again into the service_context because we will be called
    // on a static method (ThreadPingEventReady). In particular, target_process_
    // is a raw handle that is not owned by this object (it's owned by the
    // owner of this object), and we are storing it in multiple places.
    // 都放在service_context中是为了方便在静态方法ThreadPingEventReady调用
    service_context->shared_base = reinterpret_cast<char*>(shared_mem);
    service_context->channel_size = channel_size;
    service_context->channel = client_context;
    service_context->channel_buffer =
        service_context->shared_base + client_context->channel_base;
    service_context->dispatcher = call_dispatcher_;
    service_context->target_info.process = target_process_;
    service_context->target_info.process_id = target_process_id_;
    // Advance to the next channel.
    base_start += channel_size;
    // Register the ping event with the threadpool.
    // 这里通过thread_provider的接口将ping事件与ThreadPingEventReady绑定，service_context作为
    // ping事件signaled时，调用ThreadPingEventReady所携带的参数
    thread_provider_->RegisterWait(this, service_context->ping_event.Get(),
                                   ThreadPingEventReady, service_context);
  }
  if (!::DuplicateHandle(::GetCurrentProcess(), g_alive_mutex, target_process_,
                         &client_control_->server_alive,
                         SYNCHRONIZE | EVENT_MODIFY_STATE, false, 0)) {
    return false;
  }
  // This last setting indicates to the client all is setup.
  // 到此，所有的channel都实装了
  client_control_->channels_count = channel_count;
  return true;
}
```

再看ThreadPingEventReady:

```cpp
// This function gets called by a thread from the thread pool when a
// ping event fires. The context is the same as passed in the RegisterWait()
// call above.
// 客户端会把thread_provider_->RegisterWait绑定的service_context->ping_event置信
// 然后windows会回调ThreadPingEventReady，context实际上就是service_context这个ServerControl对象
void __stdcall SharedMemIPCServer::ThreadPingEventReady(void* context,
                                                        unsigned char) {
  if (!context) {
    DCHECK(false);
    return;
  }
  ServerControl* service_context = reinterpret_cast<ServerControl*>(context);//接驾
  // Since the event fired, the channel *must* be busy. Change to kAckChannel
  // while we service it.
  // 对state的修改必须是原子操作，如果当前是kBusyChannel（这个应该由client设置）
  // 就设置成kAckChannel
  LONG last_state = ::InterlockedCompareExchange(
      &service_context->channel->state, kAckChannel, kBusyChannel);
  // 如果此前的状态不是kBusyChannel，说明调用异常
  if (kBusyChannel != last_state) {
    DCHECK(false);
    return;
  }

  // Prepare the result structure. At this point we will return some result
  // even if the IPC is invalid, malformed or has no handler.
  // CrossCallReturn是在这构造的。
  CrossCallReturn call_result = {0};
  // 追根溯源，实际上是sharedMem中的一个channel，每个IPC调用用到的是共享内存分割成的
  // 众多信道中的一个，channel_buffer就是该channel的buffer起始
  void* buffer = service_context->channel_buffer;	

  // 这个就是关键call，结果存在了局部的CrossCallReturn对象中
  InvokeCallback(service_context, buffer, &call_result);

  // Copy the answer back into the channel and signal the pong event. This
  // should wake up the client so it can finish the ipc cycle.
  // 把answer CrossCallReturn对象拷贝回与client共享的buffer
  // (call_params是个ActualCallParams，内部包含一个CrossCallReturn结构)
  // 然后将pong event置signaled
  // 切换channel的state状态到kAckChannel（这里又设置了一次，可能InvokeCallback中改了该值）
  CrossCallParams* call_params = reinterpret_cast<CrossCallParams*>(buffer);
  memcpy(call_params->GetCallReturn(), &call_result, sizeof(call_result));
  ::InterlockedExchange(&service_context->channel->state, kAckChannel);
  ::SetEvent(service_context->pong_event.Get());
}
```

InvokeCallback就是分发了：

```cpp
bool SharedMemIPCServer::InvokeCallback(const ServerControl* service_context,
                                        void* ipc_buffer,
                                        CrossCallReturn* call_result) {
  // Set the default error code;
  SetCallError(SBOX_ERROR_INVALID_IPC, call_result); // 设置的是call_result的call_outcome
  uint32_t output_size = 0;
  // Parse, verify and copy the message. The handler operates on a copy
  // of the message so the client cannot play dirty tricks by changing the
  // data in the channel while the IPC is being processed.
  // server用一个CrossCallParamsEx来承载channel buffer中的数据
  // output_size会返回channel buffer中真实有效的数据尺寸（传输的内容一般没有填满channel buffer）
  std::unique_ptr<CrossCallParamsEx> params(CrossCallParamsEx::CreateFromBuffer(
      ipc_buffer, service_context->channel_size, &output_size));
  if (!params.get())
    return false;

  // 获取信道中IPC调用的tag
  uint32_t tag = params->GetTag();
  static_assert(0 == INVALID_TYPE, "incorrect type enum");
  // 构造一个IPCParams
  IPCParams ipc_params = {0};
  ipc_params.ipc_tag = tag;

  void* args[kMaxIpcParams];	// 最多有9个参数，通过GetArgs获取全部参数，一会儿展开这个函数
  if (!GetArgs(params.get(), &ipc_params, args))
    return false;

  // 构造IPCInfo，它是callback家族函数的第一个参数
  IPCInfo ipc_info = {0};
  ipc_info.ipc_tag = tag;	// IPC调用的tag
  ipc_info.client_info = &service_context->target_info;	// 传入的target pid和handle
  Dispatcher* dispatcher = service_context->dispatcher;	// 传入的dispatcher
  DCHECK(dispatcher);
  bool error = true;
  Dispatcher* handler = nullptr;

  Dispatcher::CallbackGeneric callback_generic;
  // 总算是找到上一节的答案了，在这里调用了OnMessageReady来匹配handler
  handler = dispatcher->OnMessageReady(&ipc_params, &callback_generic);
  // 如果这个dispatcher可以处理的话，那就用返回的callback_generic来处理IPC调用
  if (handler) {
    // 又是呆萌的不定参数匹配
    switch (params->GetParamsCount()) {
      case 0: {
        // Ask the IPC dispatcher if it can service this IPC.
        // 无参数的IPC调用，转成Callback0来处理，下面同理
        // 换句话说，CallbackGeneric类型只是个幌子，真正的callback参数肯定是对得上的。
        Dispatcher::Callback0 callback =
            reinterpret_cast<Dispatcher::Callback0>(callback_generic);
        // 进行调用
        if (!(handler->*callback)(&ipc_info))
          break;
        error = false;
        break;
      }
      case 1: {
        Dispatcher::Callback1 callback =
            reinterpret_cast<Dispatcher::Callback1>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0]))
          break;
        error = false;
        break;
      }
      case 2: {
        Dispatcher::Callback2 callback =
            reinterpret_cast<Dispatcher::Callback2>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1]))
          break;
        error = false;
        break;
      }
      case 3: {
        Dispatcher::Callback3 callback =
            reinterpret_cast<Dispatcher::Callback3>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2]))
          break;
        error = false;
        break;
      }
      case 4: {
        Dispatcher::Callback4 callback =
            reinterpret_cast<Dispatcher::Callback4>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2],
                                  args[3]))
          break;
        error = false;
        break;
      }
      case 5: {
        Dispatcher::Callback5 callback =
            reinterpret_cast<Dispatcher::Callback5>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2], args[3],
                                  args[4]))
          break;
        error = false;
        break;
      }
      case 6: {
        Dispatcher::Callback6 callback =
            reinterpret_cast<Dispatcher::Callback6>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2], args[3],
                                  args[4], args[5]))
          break;
        error = false;
        break;
      }
      case 7: {
        Dispatcher::Callback7 callback =
            reinterpret_cast<Dispatcher::Callback7>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2], args[3],
                                  args[4], args[5], args[6]))
          break;
        error = false;
        break;
      }
      case 8: {
        Dispatcher::Callback8 callback =
            reinterpret_cast<Dispatcher::Callback8>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2], args[3],
                                  args[4], args[5], args[6], args[7]))
          break;
        error = false;
        break;
      }
      case 9: {
        Dispatcher::Callback9 callback =
            reinterpret_cast<Dispatcher::Callback9>(callback_generic);
        if (!(handler->*callback)(&ipc_info, args[0], args[1], args[2], args[3],
                                  args[4], args[5], args[6], args[7], args[8]))
          break;
        error = false;
        break;
      }
      default: {
        NOTREACHED();
        break;
      }
    }
  }

  // 调用发生了错误的话，就要设置SBOX_ERROR_FAILED_IPC错误
  if (error) {
    if (handler)
      SetCallError(SBOX_ERROR_FAILED_IPC, call_result);	
  } else {
    // 调用成功，这时callback已经填充了ipc_info.return_info这个CrossCallReturn对象了
    memcpy(call_result, &ipc_info.return_info, sizeof(*call_result));
    SetCallSuccess(call_result);
    if (params->IsInOut()) {
      // Maybe the params got changed by the broker. We need to upadte the
      // memory section.
      // 如果有InOut型参数，那么还得把CrossCallParamsEx对象copy回channel buffer
      // 这是因为static CrossCallParamsEx::CreateFromBuffer内部对CrossCallParamsEx的make
      // 并不是直接对channel buffer进行了类型转换，而是copy了channel buffer到新new出来的空间中
      // 如果你不记得了，请回去看crosscall一篇
      memcpy(ipc_buffer, params.get(), output_size);
    }
  }

  // 该放放，好习惯
  ReleaseArgs(&ipc_params, args);

  return !error;
}
```

先看看`GetArgs`和`ReleaseArgs`：

```cpp
// IPCParams是用于在OnMessageReady中判断该种类型IPC是否是本dispatcher可以处理的参考
// dispatcher会预置很多IPCParams，而到来的IPC调用就用它的IPCParams来lookup
//
// Releases memory allocated for IPC arguments, if needed.
void ReleaseArgs(const IPCParams* ipc_params, void* args[kMaxIpcParams]) {
  for (size_t i = 0; i < kMaxIpcParams; i++) {
    switch (ipc_params->args[i]) {
      // 其实只有两种要delete资源，一种是字符串，一种是INOUTPTR
      case WCHAR_TYPE: {
        delete reinterpret_cast<base::string16*>(args[i]);
        args[i] = nullptr;
        break;
      }
      case INOUTPTR_TYPE: {
        delete reinterpret_cast<CountedBuffer*>(args[i]);
        args[i] = nullptr;
        break;
      }
      default:
        break;
    }
  }
}

// Fills up the list of arguments (args and ipc_params) for an IPC call.
bool GetArgs(CrossCallParamsEx* params,
             IPCParams* ipc_params,
             void* args[kMaxIpcParams]) {
  if (kMaxIpcParams < params->GetParamsCount())
    return false;

  for (uint32_t i = 0; i < params->GetParamsCount(); i++) {
    uint32_t size;
    ArgType type;
    // 循环调用GetRawParameter
    args[i] = params->GetRawParameter(i, &size, &type);
    if (args[i]) {
      ipc_params->args[i] = type;
      // 根据type，进行args[i]的填充
      switch (type) {
        case WCHAR_TYPE: {
          std::unique_ptr<base::string16> data(new base::string16);//这种情况要new 字符串
          if (!params->GetParameterStr(i, data.get())) {
            args[i] = 0;
            ReleaseArgs(ipc_params, args);
            return false;
          }
          args[i] = data.release();
          break;
        }
        case UINT32_TYPE: {
          uint32_t data;
          if (!params->GetParameter32(i, &data)) {
            ReleaseArgs(ipc_params, args);
            return false;
          }
          IPCInt ipc_int(data);
          args[i] = ipc_int.AsVoidPtr();
          break;
        }
        case VOIDPTR_TYPE: {
          void* data;
          if (!params->GetParameterVoidPtr(i, &data)) {
            ReleaseArgs(ipc_params, args);
            return false;
          }
          args[i] = data;
          break;
        }
        case INOUTPTR_TYPE: {
          if (!args[i]) {
            ReleaseArgs(ipc_params, args);
            return false;
          }
          CountedBuffer* buffer = new CountedBuffer(args[i], size);	// 这种情况是要new CountedBuffer的
          args[i] = buffer;
          break;
        }
        default:
          break;
      }
    }
  }
  return true;
}
```

最后看看`MakeEvents`：

```cpp
bool SharedMemIPCServer::MakeEvents(base::win::ScopedHandle* server_ping,
                                    base::win::ScopedHandle* server_pong,
                                    HANDLE* client_ping,
                                    HANDLE* client_pong) {
  // Note that the IPC client has no right to delete the events. That would
  // cause problems. The server *owns* the events.
  // event的拥有者是Server端，client无权删除
  const DWORD kDesiredAccess = SYNCHRONIZE | EVENT_MODIFY_STATE;

  // The events are auto reset, and start not signaled.
  // 实际上就是简单的CreateEventW封装，因为client和server各持有一个handle，所以要DuplicateHandle
  // 给了target进程也就是client端的event句柄
  server_ping->Set(::CreateEventW(nullptr, false, false, nullptr));
  if (!::DuplicateHandle(::GetCurrentProcess(), server_ping->Get(),
                         target_process_, client_ping, kDesiredAccess, false,
                         0)) {
    return false;
  }

  server_pong->Set(::CreateEventW(nullptr, false, false, nullptr));
  if (!::DuplicateHandle(::GetCurrentProcess(), server_pong->Get(),
                         target_process_, client_pong, kDesiredAccess, false,
                         0)) {
    return false;
  }
  return true;
}
```

到此，server的流程大体清楚了，一个疑点就是`client_control_`这个成员，根据析构的处理，这理应是个`MapViewOfFile`，但server端却不见它创建的踪影，而是直接使用，看起来共享内存的创建由client控制。

## Client

目前已经搞清楚了这部分的设计，是时候看看客户端实现了。

```cpp
// IPC transport implementation that uses shared memory.
// This is the client side
//
// The shared memory is divided on blocks called channels, and potentially
// it can perform as many concurrent IPC calls as channels. The IPC over
// each channel is strictly synchronous for the client.
// 共享内存划分成多个channel，这就可以并发处理IPC调用
// 
// Each channel as a channel control section associated with. Each control
// section has two kernel events (known as ping and pong) and a integer
// variable that maintains a state
//
// this is the state diagram of a channel:
//
//                   locked                in service
//     kFreeChannel---------->BusyChannel-------------->kAckChannel
//          ^                                                 |
//          |_________________________________________________|
//                             answer ready
//
// The protocol is as follows:
//   1) client finds a free channel: state = kFreeChannel
//   2) does an atomic compare-and-swap, now state = BusyChannel
//   3) client writes the data into the channel buffer
//   4) client signals the ping event and waits (blocks) on the pong event
//   5) eventually the server signals the pong event
//   6) the client awakes and reads the answer from the same channel
//   7) the client updates its InOut parameters with the new data from the
//      shared memory section.
//   8) the client atomically sets the state = kFreeChannel
//
//  In the shared memory the layout is as follows:
//
//    [ channel count    ]
//    [ channel control 0]
//    [ channel control 1]
//    [ channel control N]
//    [ channel buffer 0 ] 1024 bytes
//    [ channel buffer 1 ] 1024 bytes
//    [ channel buffer N ] 1024 bytes
//
// By default each channel buffer is 1024 bytes
```

client端所做的事，上面起始阐释的已经很清楚了，总结一下就是：

1. client想要发IPC调用时，先要在共享内存中找到一个free态的channel，并置为busy态
2. client把数据写入到channel buffer中，置信ping事件并在pong事件等待，以此做与server端的同步
3. server收到ping置信后，会进行我们前面剖析的流程，最终填好channel buffer并置信pong事件，channel置为ack态
4. client设置channel状态为free态

### 共享内存布局

另一方面，共享内存的结构其实之前已经隐隐约约说过了，它是个flexible的层级结构。

```cpp
// shared memory起始，一层结构
struct IPCControl {
  // total number of channels available, some might be busy at a given time
  size_t channels_count;
  // handle to a shared mutex to detect when the server is dead
  HANDLE server_alive;
  // array of channel control structures
  ChannelControl channels[1];
};

// 二层结构，一层的channels_count决定了它的数量，附着在一层数据正后方
// the channel control structure
struct ChannelControl {
  // points to be beginning of the channel buffer, where data goes
  size_t channel_base;
  // maintains the state from the ChannelState enumeration
  volatile LONG state;
  // the ping event is signaled by the client when the IPC data is ready on
  // the buffer
  HANDLE ping_event;
  // the client waits on the pong event for the IPC answer back
  HANDLE pong_event;
  // the IPC unique identifier
  uint32_t ipc_tag;
};

// 三层结构就是真实的channel buffer了，每个channel buffer大小被sandbox硬编码成了1024
// 每有一个ChannelControl，就有一个channel buffer，它的偏移由二层结构的channel_base索引
```

### `SharedMemIPCClient`

头定义：

```cpp
// the actual shared memory IPC implementation class. This object is designed
// to be lightweight so it can be constructed on-site (at the calling place)
// wherever an IPC call is needed.
// 设计成了一个轻量对象，可以在IPC调用发起时，在发端简单的构造出来
class SharedMemIPCClient {
 public:
  // Creates the IPC client.
  // as parameter it takes the base address of the shared memory
  // 显式声明，对于这种单简单类型参数的构造器都要设置explicit，防止编译器自作聪明的隐式转换
  // 可以看出来shared_mem也并不是Client控制的
  explicit SharedMemIPCClient(void* shared_mem);

  // locks a free channel and returns the channel buffer memory base. This call
  // blocks until there is a free channel
  // 占坑，找个free态channel，如果没有的话会阻塞
  void* GetBuffer();

  // releases the lock on the channel, for other to use. call this if you have
  // called GetBuffer and you want to abort but have not called yet DoCall()
  // 释放channel的锁，使用情景在你通过GetBuffer获得了channel但却在调用DoCall()前想要终止
  void FreeBuffer(void* buffer);

  // Performs the actual IPC call.
  // params: The blob of packed input parameters.
  // answer: upon IPC completion, it contains the server answer to the IPC.
  // If the return value is not SBOX_ERROR_CHANNEL_ERROR, the caller has to free
  // the channel.
  // returns ALL_OK if the IPC mechanism successfully delivered. You still need
  // to check on the answer structure to see the actual IPC result.
  // 这个DoCall就是关键的IPC调用了，这个接口实际上在crosscall中就已经看过了，当时比较奇怪的是
  // CrossCallParams内部已经有CrossCallReturn了，为什么还要额外传一个answer
  ResultCode DoCall(CrossCallParams* params, CrossCallReturn* answer);

 private:
  // Returns the index of the first free channel. It sets 'severe_failure'
  // to true if there is an unrecoverable error that does not allow to
  // find a channel.
  // 这货一看就是给GetBuffer用的内部helper函数
  size_t LockFreeChannel(bool* severe_failure);
  // Return the channel index given the address of the buffer.
  // 由buffer的address反推channel的索引
  size_t ChannelIndexFromBuffer(const void* buffer);
  IPCControl* control_;	// client端也是这个结构，指向共享内存起始
  // point to the first channel base
  char* first_base_;	// 三级channel结构的起始地址
};
```

构造器：

```cpp
// The constructor simply casts the shared memory to the internal
// structures. This is a cheap step that is why this IPC object can
// and should be constructed per call.
// 共享内存也不是client维护的，而是外部传过来的。
SharedMemIPCClient::SharedMemIPCClient(void* shared_mem)
    : control_(reinterpret_cast<IPCControl*>(shared_mem)) {
  first_base_ =
      reinterpret_cast<char*>(shared_mem) + control_->channels[0].channel_base;
  // There must be at least one channel.
  DCHECK(0 != control_->channels_count);
}
```

直观的两个private函数：

```cpp
// Locking a channel is a simple as looping over all the channels
// looking for one that is has state = kFreeChannel and atomically
// swapping it to kBusyChannel.
// If there is no free channel, then we must back off so some other
// thread makes progress and frees a channel. To back off we sleep.
size_t SharedMemIPCClient::LockFreeChannel(bool* severe_failure) {
  if (0 == control_->channels_count) {
    *severe_failure = true;
    return 0;
  }
  //对channel做遍历，找第一个kFreeChannel态的channel，如果找到就置为busy态
  ChannelControl* channel = control_->channels;
  do {
    for (size_t ix = 0; ix != control_->channels_count; ++ix) {
      if (kFreeChannel == ::InterlockedCompareExchange(
                              &channel[ix].state, kBusyChannel, kFreeChannel)) {
        *severe_failure = false;
        return ix;
      }
    }
    // We did not find any available channel, maybe the server is dead.
    // 如果当前没有空闲channel的话，先判断一下server是否还在，挂了就直接failure吧，也没有IPC的必要了
    // 否则就一直轮询
    DWORD wait =
        ::WaitForSingleObject(control_->server_alive, kIPCWaitTimeOut2);
    if (WAIT_TIMEOUT != wait) {
      // The server is dead and we outlive it enough to get in trouble.
      *severe_failure = true;
      return 0;
    }
  } while (true);
}

// Find out which channel we are from the pointer returned by GetBuffer.
// 简单的四则运算，基操，勿6
size_t SharedMemIPCClient::ChannelIndexFromBuffer(const void* buffer) {
  ptrdiff_t d = reinterpret_cast<const char*>(buffer) - first_base_;
  size_t num = d / kIPCChannelSize;
  DCHECK_LT(num, control_->channels_count);
  return (num);
}
```

再看对buffer的处理，应该是用于test的封装：

```cpp
// Get the base of the data buffer of the channel; this is where the input
// parameters get serialized. Since they get serialized directly into the
// channel we avoid one copy.
// 获取一个free态channel
void* SharedMemIPCClient::GetBuffer() {
  bool failure = false;
  size_t ix = LockFreeChannel(&failure);
  // 这种表示server已经挂了
  if (failure)
    return nullptr;
  // 基本四则运算
  return reinterpret_cast<char*>(control_) +
         control_->channels[ix].channel_base;
}

// If we need to cancel an IPC before issuing DoCall
// our client should call FreeBuffer with the same pointer
// returned by GetBuffer.
// 这里的free不是释放channel buffer，而是把channel置为free态，表示可用
void SharedMemIPCClient::FreeBuffer(void* buffer) {
  size_t num = ChannelIndexFromBuffer(buffer);
  ChannelControl* channel = control_->channels;
  LONG result = ::InterlockedExchange(&channel[num].state, kFreeChannel);
  DCHECK_NE(kFreeChannel, static_cast<ChannelState>(result));
}
```

再看与server联系紧密的`DoCall`:

```cpp
// Do the IPC. At this point the channel should have already been
// filled with the serialized input parameters.
// We follow the pattern explained in the header file.
// params和answer都是外部传入的，我们毕竟只是个驱动器，承载什么消息是驱动者决定的
ResultCode SharedMemIPCClient::DoCall(CrossCallParams* params,
                                      CrossCallReturn* answer) {
  // server要健在
  if (!control_->server_alive)
    return SBOX_ERROR_CHANNEL_ERROR;

  // channel的buffer就是params对象本身（params->GetBuffer返回this）
  // channel buffer承载了crosscall_client中看到的输入结构
  // 外部在调用该函数时，应该已经把params放入了shareMem的某个channel中，这里素质二连算出num，主要是为了直接操纵channel[num]来把参数导入
  size_t num = ChannelIndexFromBuffer(params->GetBuffer());
  ChannelControl* channel = control_->channels;
  // Note that the IPC tag goes outside the buffer as well inside
  // the buffer. This should enable the server to prioritize based on
  // IPC tags without having to de-serialize the entire message.
  channel[num].ipc_tag = params->GetTag();

  // Wait for the server to service this IPC call. After kIPCWaitTimeOut1
  // we check if the server_alive mutex was abandoned which will indicate
  // that the server has died.

  // While the atomic signaling and waiting is not a requirement, it
  // is nice because we save a trip to kernel.
  // 填充好channelcontrol的参数后，就可以直接signaled ping通知server了
  // 然后在pong上等待，等待时间kIPCWaitTimeOut1
  DWORD wait =
      ::SignalObjectAndWait(channel[num].ping_event, channel[num].pong_event,
                            kIPCWaitTimeOut1, false);
  if (WAIT_TIMEOUT == wait) {
    // The server is taking too long. Enter a loop were we check if the
    // server_alive mutex has been abandoned which would signal a server crash
    // or else we keep waiting for a response.
    // 如果超时了，那么得探测一下server是否还存活，是否只是因为繁忙而暂时没处理IPC
    while (true) {
      wait = ::WaitForSingleObject(control_->server_alive, 0);
      if (WAIT_TIMEOUT == wait) {
        // Server seems still alive. We already signaled so here we just wait.
        wait = ::WaitForSingleObject(channel[num].pong_event, kIPCWaitTimeOut1);
        if (WAIT_OBJECT_0 == wait) {
          // The server took a long time but responded.
          break;
        } else if (WAIT_TIMEOUT == wait) {
          continue;
        } else {
          return SBOX_ERROR_CHANNEL_ERROR;
        }
      } else {
        // The server has crashed and windows has signaled the mutex as
        // abandoned.
        // server崩溃了，玩个J8
        ::InterlockedExchange(&channel[num].state, kAbandonedChannel);
        control_->server_alive = 0;
        return SBOX_ERROR_CHANNEL_ERROR;
      }
    }
  } else if (WAIT_OBJECT_0 != wait) {
    // Probably the server crashed before the kIPCWaitTimeOut1 occurred.
    return SBOX_ERROR_CHANNEL_ERROR;
  }
	
  // 按常规操作，WAIT_OBJECT_0这个返回值就对了，channel buffer实际上就是params对象
  // server处理后params已经被重置了，其中params的CrossCallReturn对象被填充好，
  // 如果有INOUT型参数的话，那么params本身的输入参数也被修改了。
  // Client端把params的CrossCallReturn对象抽离出来，拷贝到了传入的answer参数
  // 到这儿也就明白了为什么此前参数中既然已有了CrossCallParams还要再传一个CrossCallReturn
  // 原来只是为了纯粹的剥离params，二者实际上是同一个东西，DoCall返回后answer还有用
  // The server has returned an answer, copy it and free the channel.
  memcpy(answer, params->GetCallReturn(), sizeof(CrossCallReturn));

  // Return the IPC state It can indicate that while the IPC has
  // completed some error in the Broker has caused to not return valid
  // results.
  return answer->call_outcome;	// 这个是本次IPC调用的成功或失败的状态码
}
```

到此，也就明白了`SharedMemIPC`一种基于共享内存的操纵器，现在的问题就在于client和server两端`SharedMemIPC`的操纵者是如何操纵的。

由于当前我们还没有见过target进程完整的时间线，所以client的操纵暂时不关心。我们找找server的时间线。

最终在`BrokerServicesBase::SpawnTarget => PolicyBase::AddTarget => TargetProcess::Init`中找到了蛛丝马迹：
```cpp
// Construct the IPC server and the IPC dispatcher. When the target does
// an IPC it will eventually call the dispatcher.
// TargetProcess对象会挂到broker创建出来的PolicyBase对象中管理
// 一定要搞清楚TargetProcess是broker的，不是target的，它是broker中用于表示target进程的结构
// 上层传入的是：
// ret = target->Init(dispatcher_.get(), policy_, kIPCMemSize, kPolMemSize, &win_error);
// dispatcher_就是用于在OnMessageReady中匹配参数的那货
// policy_是low-level-policy的buffer，这个暂时不关心
// kIPCMemSize是SharedMemIPC所用buffer的尺寸，这里设置成了2页
// kPolMemSize是low-level-policy的buffer尺寸，设置成了14页，这个暂时不关心
ResultCode TargetProcess::Init(Dispatcher* ipc_dispatcher,
                               void* policy,
                               uint32_t shared_IPC_size,
                               uint32_t shared_policy_size,
                               DWORD* win_error) {
  // We need to map the shared memory on the target. This is necessary for
  // any IPC that needs to take place, even if the target has not yet hit
  // the main( ) function or even has initialized the CRT. So here we set
  // the handle to the shared section. The target on the first IPC must do
  // the rest, which boils down to calling MapViewofFile()

  // We use this single memory pool for IPC and for policy.
  DWORD shared_mem_size =
      static_cast<DWORD>(shared_IPC_size + shared_policy_size);
  // 总算找到你了，这里是把SharedMemIPC和low-level-policy的共享内存一并创建出来
  shared_section_.Set(::CreateFileMappingW(INVALID_HANDLE_VALUE, nullptr,
                                           PAGE_READWRITE | SEC_COMMIT, 0,
                                           shared_mem_size, nullptr));
  if (!shared_section_.IsValid()) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_CREATE_FILE_MAPPING;
  }

  // 设置共享内存的权限：读写查询
  DWORD access = FILE_MAP_READ | FILE_MAP_WRITE | SECTION_QUERY;
  HANDLE target_shared_section;
  // 把broker进程的共享内存句柄复制给target进程
  if (!::DuplicateHandle(::GetCurrentProcess(), shared_section_.Get(),
                         sandbox_process_info_.process_handle(),
                         &target_shared_section, access, false, 0)) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_DUPLICATE_SHARED_SECTION;
  }

  // 文件映射，CreateFileMapping+MapViewOfFile共享内存素质二连
  void* shared_memory = ::MapViewOfFile(
      shared_section_.Get(), FILE_MAP_WRITE | FILE_MAP_READ, 0, 0, 0);
  if (!shared_memory) {
    *win_error = ::GetLastError();
    return SBOX_ERROR_MAP_VIEW_OF_SHARED_SECTION;
  }

  // 这个暂时不关心，把low-level-policy通过共享内存的方式拷贝给target
  CopyPolicyToTarget(policy, shared_policy_size,
                     reinterpret_cast<char*>(shared_memory) + shared_IPC_size);

  ResultCode ret;
  // Set the global variables in the target. These are not used on the broker.
  // target_shared_section是为target进程复制出来的句柄，通过TransferVariable的方式把它的值
  // 赋值给了target进程中的一个全局变量g_shared_section
  // 这个TransferVariable在分析TargetProcess时就看过了
  // 实际上就是LoadLibrary+GetProcAddress+WriteProcessMemory素质3连
  g_shared_section = target_shared_section;
  ret = TransferVariable("g_shared_section", &g_shared_section,
                         sizeof(g_shared_section));
  // broker本身不用这货
  g_shared_section = nullptr;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }
  // 如法炮制，把shared_IPC_size也赋值给target进程的g_shared_IPC_size全局变量
  g_shared_IPC_size = shared_IPC_size;
  ret = TransferVariable("g_shared_IPC_size", &g_shared_IPC_size,
                         sizeof(g_shared_IPC_size));
  g_shared_IPC_size = 0;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }
  // 同上
  g_shared_policy_size = shared_policy_size;
  ret = TransferVariable("g_shared_policy_size", &g_shared_policy_size,
                         sizeof(g_shared_policy_size));
  g_shared_policy_size = 0;
  if (SBOX_ALL_OK != ret) {
    *win_error = ::GetLastError();
    return ret;
  }

  // buffer已经准备好了，这里就new出了SharedMemIPCServer
  // 智能指针ipc_server_由TargetProcess对象管理
  ipc_server_.reset(new SharedMemIPCServer(
      sandbox_process_info_.process_handle(),
      sandbox_process_info_.process_id(), thread_pool_, ipc_dispatcher));

  if (!ipc_server_->Init(shared_memory, shared_IPC_size, kIPCChannelSize))
    return SBOX_ERROR_NO_SPACE;

  // After this point we cannot use this handle anymore.
  ::CloseHandle(sandbox_process_info_.TakeThreadHandle());

  return SBOX_ALL_OK;
}
```

看到这里可以发现，server匹配用的`Dispatcher`对象不是`TargetProcess`控制的，而是一早就创建好了的，`TargetProcess`内部创建了`SharedMemIPC`机制的共享内存并new出了`SharedMemIPCServer`对象。

向上翻，可以发现`Dispatcher`是`PolicyBase`对象控制的成员，这也是合理的，因为共享内存IPC交互是进程之间的事，理应进程对象来管理，但匹配处理的事宜就应该由Policy来管理，所以`Dispatcher`对象作为`PolicyBase`的一员也就合情合理了。

我们找找`Dispatcher`在哪里创建：

```cpp
PolicyBase::PolicyBase()
    : ref_count(1),
      lockdown_level_(USER_LOCKDOWN),
      initial_level_(USER_LOCKDOWN),
      job_level_(JOB_LOCKDOWN),
      ui_exceptions_(0),
      memory_limit_(0),
      use_alternate_desktop_(false),
      use_alternate_winstation_(false),
      file_system_init_(false),
      relaxed_interceptions_(true),
      stdout_handle_(INVALID_HANDLE_VALUE),
      stderr_handle_(INVALID_HANDLE_VALUE),
      integrity_level_(INTEGRITY_LEVEL_LAST),
      delayed_integrity_level_(INTEGRITY_LEVEL_LAST),
      mitigations_(0),
      delayed_mitigations_(0),
      is_csrss_connected_(true),
      policy_maker_(nullptr),
      policy_(nullptr),
      lowbox_sid_(nullptr),
      lockdown_default_dacl_(false),
      enable_opm_redirection_(false) {
  ::InitializeCriticalSection(&lock_);
  // 还记得之前在分析Dispatcher类的时候，发现Dispatcher只是个抽象基类的结论吗？
  // 没错，Dispatcher本身没有实际意义，只是个框架，在使用时，PolicyBase使用了具有实际意义的
  // TopLevelDispatcher，那么它究竟是什么？
  dispatcher_.reset(new TopLevelDispatcher(this));
}
```

我们好奇的展开`TopLevelDispatcher`看看，注意它传入了`PolicyBase`对象本身。

### `TopLevelDispatcher`

想要理解一个类的构造，先得了解它的类头定义：

```cpp
// Top level dispatcher which hands requests to the appropriate service
// dispatchers.
// 看起来是个顶层的dispatcher，维护了多个子系统dispatcher，用于按类别分发IPC请求
class TopLevelDispatcher : public Dispatcher {
 public:
  // |policy| must outlive this class.
  explicit TopLevelDispatcher(PolicyBase* policy);
  ~TopLevelDispatcher() override;

  Dispatcher* OnMessageReady(IPCParams* ipc,
                             CallbackGeneric* callback) override;
  // 还记得Dispatcher类的该函数吗，它当初的注释是这样的：
  // Called when a target proces is created, to setup the interceptions related
  // with the given service (IPC).
  // Interception看起来是施加给IPC请求的一个拦截，但它如何奏效，得在分析Interception的机制后才会明白
  bool SetupService(InterceptionManager* manager, int service) override;

 private:
  // Test IPC provider.
  bool Ping(IPCInfo* ipc, void* cookie);

  // Returns a dispatcher from ipc_targets_.
  // 想必是OnMessageReady内部会根据IPCParams携带的ipc_tag来找到具体的下层Dispatcher
  Dispatcher* GetDispatcher(int ipc_tag);

  PolicyBase* policy_;	//关联的PolicyBase对象，这个我们已经看到了在构造器中会传入
  // 各种下层Dispatcher，每个子系统对应一个Dispatcher
  std::unique_ptr<Dispatcher> filesystem_dispatcher_;
  std::unique_ptr<Dispatcher> named_pipe_dispatcher_;
  std::unique_ptr<Dispatcher> thread_process_dispatcher_;
  std::unique_ptr<Dispatcher> sync_dispatcher_;
  std::unique_ptr<Dispatcher> registry_dispatcher_;
  std::unique_ptr<Dispatcher> handle_dispatcher_;
  std::unique_ptr<Dispatcher> process_mitigations_win32k_dispatcher_;
  Dispatcher* ipc_targets_[IPC_LAST_TAG];	// 下层分发器指针数组,IPC_LAST_TAG是ipc tag的数量
  // 每个IPC请求都对应到一个具体的下层子系统Dispatcher，可以重复
  DISALLOW_COPY_AND_ASSIGN(TopLevelDispatcher);
};
```

IPC的tag实际上在ipc_tags.h给出了定义：

```cpp
enum {
  IPC_UNUSED_TAG = 0,
  IPC_PING1_TAG,  // Takes a cookie in parameters and returns the cookie
                  // multiplied by 2 and the tick_count. Used for testing only.
  IPC_PING2_TAG,  // Takes an in/out cookie in parameters and modify the cookie
                  // to be multiplied by 3. Used for testing only.
  IPC_NTCREATEFILE_TAG,
  IPC_NTOPENFILE_TAG,
  IPC_NTQUERYATTRIBUTESFILE_TAG,
  IPC_NTQUERYFULLATTRIBUTESFILE_TAG,
  IPC_NTSETINFO_RENAME_TAG,
  IPC_CREATENAMEDPIPEW_TAG,
  IPC_NTOPENTHREAD_TAG,
  IPC_NTOPENPROCESS_TAG,
  IPC_NTOPENPROCESSTOKEN_TAG,
  IPC_NTOPENPROCESSTOKENEX_TAG,
  IPC_CREATEPROCESSW_TAG,
  IPC_CREATEEVENT_TAG,
  IPC_OPENEVENT_TAG,
  IPC_NTCREATEKEY_TAG,
  IPC_NTOPENKEY_TAG,
  IPC_GDI_GDIDLLINITIALIZE_TAG,
  IPC_GDI_GETSTOCKOBJECT_TAG,
  IPC_USER_REGISTERCLASSW_TAG,
  IPC_CREATETHREAD_TAG,
  IPC_USER_ENUMDISPLAYMONITORS_TAG,
  IPC_USER_ENUMDISPLAYDEVICES_TAG,
  IPC_USER_GETMONITORINFO_TAG,
  IPC_GDI_CREATEOPMPROTECTEDOUTPUTS_TAG,
  IPC_GDI_GETCERTIFICATE_TAG,
  IPC_GDI_GETCERTIFICATESIZE_TAG,
  IPC_GDI_DESTROYOPMPROTECTEDOUTPUT_TAG,
  IPC_GDI_CONFIGUREOPMPROTECTEDOUTPUT_TAG,
  IPC_GDI_GETOPMINFORMATION_TAG,
  IPC_GDI_GETOPMRANDOMNUMBER_TAG,
  IPC_GDI_GETSUGGESTEDOPMPROTECTEDOUTPUTARRAYSIZE_TAG,
  IPC_GDI_SETOPMSIGNINGKEYANDSEQUENCENUMBERS_TAG,
  IPC_LAST_TAG
};
```

那么看看构造器

```cpp
TopLevelDispatcher::TopLevelDispatcher(PolicyBase* policy) : policy_(policy) {
  // Initialize the IPC dispatcher array.
  memset(ipc_targets_, 0, sizeof(ipc_targets_));
  Dispatcher* dispatcher;

  // new出多个子系统的Dispatcher，指派对应的那些IPC请求
  dispatcher = new FilesystemDispatcher(policy_);
  ipc_targets_[IPC_NTCREATEFILE_TAG] = dispatcher;
  ipc_targets_[IPC_NTOPENFILE_TAG] = dispatcher;
  ipc_targets_[IPC_NTSETINFO_RENAME_TAG] = dispatcher;
  ipc_targets_[IPC_NTQUERYATTRIBUTESFILE_TAG] = dispatcher;
  ipc_targets_[IPC_NTQUERYFULLATTRIBUTESFILE_TAG] = dispatcher;
  filesystem_dispatcher_.reset(dispatcher);

  dispatcher = new NamedPipeDispatcher(policy_);
  ipc_targets_[IPC_CREATENAMEDPIPEW_TAG] = dispatcher;
  named_pipe_dispatcher_.reset(dispatcher);

  dispatcher = new ThreadProcessDispatcher(policy_);
  ipc_targets_[IPC_NTOPENTHREAD_TAG] = dispatcher;
  ipc_targets_[IPC_NTOPENPROCESS_TAG] = dispatcher;
  ipc_targets_[IPC_CREATEPROCESSW_TAG] = dispatcher;
  ipc_targets_[IPC_NTOPENPROCESSTOKEN_TAG] = dispatcher;
  ipc_targets_[IPC_NTOPENPROCESSTOKENEX_TAG] = dispatcher;
  ipc_targets_[IPC_CREATETHREAD_TAG] = dispatcher;
  thread_process_dispatcher_.reset(dispatcher);

  dispatcher = new SyncDispatcher(policy_);
  ipc_targets_[IPC_CREATEEVENT_TAG] = dispatcher;
  ipc_targets_[IPC_OPENEVENT_TAG] = dispatcher;
  sync_dispatcher_.reset(dispatcher);

  dispatcher = new RegistryDispatcher(policy_);
  ipc_targets_[IPC_NTCREATEKEY_TAG] = dispatcher;
  ipc_targets_[IPC_NTOPENKEY_TAG] = dispatcher;
  registry_dispatcher_.reset(dispatcher);

  dispatcher = new ProcessMitigationsWin32KDispatcher(policy_);
  ipc_targets_[IPC_GDI_GDIDLLINITIALIZE_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETSTOCKOBJECT_TAG] = dispatcher;
  ipc_targets_[IPC_USER_REGISTERCLASSW_TAG] = dispatcher;
  ipc_targets_[IPC_USER_ENUMDISPLAYMONITORS_TAG] = dispatcher;
  ipc_targets_[IPC_USER_ENUMDISPLAYDEVICES_TAG] = dispatcher;
  ipc_targets_[IPC_USER_GETMONITORINFO_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_CREATEOPMPROTECTEDOUTPUTS_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETCERTIFICATE_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETCERTIFICATESIZE_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_DESTROYOPMPROTECTEDOUTPUT_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_CONFIGUREOPMPROTECTEDOUTPUT_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETOPMINFORMATION_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETOPMRANDOMNUMBER_TAG] = dispatcher;
  ipc_targets_[IPC_GDI_GETSUGGESTEDOPMPROTECTEDOUTPUTARRAYSIZE_TAG] =
      dispatcher;
  ipc_targets_[IPC_GDI_SETOPMSIGNINGKEYANDSEQUENCENUMBERS_TAG] = dispatcher;
  process_mitigations_win32k_dispatcher_.reset(dispatcher);
}
```

构造器做了IPC请求的分类，而在`OnMessageReady`中则只是对分发的包装：

```cpp
// When an IPC is ready in any of the targets we get called. We manage an array
// of IPC dispatchers which are keyed on the IPC tag so we normally delegate
// to the appropriate dispatcher unless we can handle the IPC call ourselves.
Dispatcher* TopLevelDispatcher::OnMessageReady(IPCParams* ipc,
                                               CallbackGeneric* callback) {
  DCHECK(callback);
  static const IPCParams ping1 = {IPC_PING1_TAG, {UINT32_TYPE}};
  static const IPCParams ping2 = {IPC_PING2_TAG, {INOUTPTR_TYPE}};

  if (ping1.Matches(ipc) || ping2.Matches(ipc)) {
    *callback = reinterpret_cast<CallbackGeneric>(
        static_cast<Callback1>(&TopLevelDispatcher::Ping));
    return this;
  }

  // 父类指针指向子类对象
  Dispatcher* dispatcher = GetDispatcher(ipc->ipc_tag);
  if (!dispatcher) {
    NOTREACHED();
    return nullptr;
  }
  // 调用真正的子系统Dispatcher派生类来处理
  return dispatcher->OnMessageReady(ipc, callback);
}
```

到此，我们还剩下两个疑点：

1. `SetupService`具体作用是什么，如何与Interception联系并工作的？
2. 这些具体的子系统Dispatcher派生类是如何工作的？

关于第一点，日后在分析Interception时自然知晓。而第二点则在日后分析完Interception+Dispatcher+Policy三大组成基础设施后，逐一分析每个子系统的部署。

## 管中窥豹

当然，sandbox有很多单元测试代码，透过lpc_unittest.cc中对IPC的测试代码可以窥探一二。

```cpp
// Creates a server thread that answers the IPC so slow that is guaranteed to
// trigger the time-out code path in the client. A second thread is created
// to hold locked the server_alive mutex: this signals the client that the
// server is not dead and it retries the wait.
TEST(IPCTest, ClientSlowServer) {
  size_t base_start = 0;
  IPCControl* client_control =
      MakeChannels(kIPCChannelSize, 4096 * 2, &base_start);	// 实际上并没有共享，只是new了一片空间
  FixChannels(client_control, base_start, kIPCChannelSize, FIX_PONG_NOT_READY);
  client_control->server_alive = ::CreateMutex(nullptr, false, nullptr);

  char* mem = reinterpret_cast<char*>(client_control);
  SharedMemIPCClient client(mem);	// 局部做出一个SharedMemIPCClient对象

  ServerEvents events = {0};
  events.ping = client_control->channels[0].ping_event;
  events.pong = client_control->channels[0].pong_event;
  events.state = &client_control->channels[0].state;

  // 因为没有共享内存，不是真正的进程间通信，所以只是用同一进程中线程间通信来测试功能代码
  // 这个是回复IPC调用的线程
  HANDLE t1 =
      ::CreateThread(nullptr, 0, SlowResponseServer, &events, 0, nullptr);
  ASSERT_TRUE(t1);
  ::CloseHandle(t1);

  ServerEvents events2 = {0};
  events2.pong = events.pong;
  events2.mutex = client_control->server_alive;

  // 这个是控制client_control->server_alive锁的线程
  HANDLE t2 =
      ::CreateThread(nullptr, 0, MainServerThread, &events2, 0, nullptr);
  ASSERT_TRUE(t2);
  ::CloseHandle(t2);

  ::Sleep(1);

  // 这里获取了channel，并填充了CrossCallParamsMock，它是个测试用的CrossCallParams子类
  /*
    class CrossCallParamsMock : public CrossCallParams {
     public:
      CrossCallParamsMock(uint32_t tag, uint32_t params_count)
          : CrossCallParams(tag, params_count) {}
    };
  */
  void* buff0 = client.GetBuffer();
  uint32_t tag = 4321;
  CrossCallReturn answer;
  CrossCallParamsMock* params1 = new (buff0) CrossCallParamsMock(tag, 1);
  FakeOkAnswerInChannel(buff0);

  // 调用了DoCall，印证了channel buffer已经填充完毕的想法
  ResultCode result = client.DoCall(params1, &answer);/
  if (SBOX_ERROR_CHANNEL_ERROR != result)
    client.FreeBuffer(buff0);

  // 判断一下这些值是否正确
  EXPECT_TRUE(SBOX_ALL_OK == result);
  EXPECT_EQ(tag, client_control->channels[0].ipc_tag);
  EXPECT_EQ(kFreeChannel, client_control->channels[0].state);

  CloseChannelEvents(client_control);
  ::CloseHandle(client_control->server_alive);
  delete[] reinterpret_cast<char*>(client_control);
}
```

展开`MakeChannels`:

```cpp
// Helper function to make the fake shared memory with some
// basic elements initialized.
IPCControl* MakeChannels(size_t channel_size,
                         size_t total_shared_size,
                         size_t* base_start) {
  // Allocate memory
  char* mem = new char[total_shared_size];// new一片内存空间，部署好IPCControl
  memset(mem, 0, total_shared_size);
  // Calculate how many channels we can fit in the shared memory.
  total_shared_size -= offsetof(IPCControl, channels);
  size_t channel_count =
      total_shared_size / (sizeof(ChannelControl) + channel_size);
  // Calculate the start of the first channel.
  *base_start =
      (sizeof(ChannelControl) * channel_count) + offsetof(IPCControl, channels);
  // Setup client structure.
  IPCControl* client_control = reinterpret_cast<IPCControl*>(mem);
  client_control->channels_count = channel_count;
  return client_control;
}

// 创建ping/pong事件
void FixChannels(IPCControl* client_control,
                 size_t base_start,
                 size_t channel_size,
                 TestFixMode mode) {
  for (size_t ix = 0; ix != client_control->channels_count; ++ix) {
    ChannelControl& channel = client_control->channels[ix];
    channel.channel_base = base_start;
    channel.state = kFreeChannel;
    if (mode != FIX_NO_EVENTS) {
      bool signaled = (FIX_PONG_READY == mode) ? true : false;
      channel.ping_event = ::CreateEventW(nullptr, false, false, nullptr);
      channel.pong_event = ::CreateEventW(nullptr, false, signaled, nullptr);
    }
    base_start += channel_size;
  }
}
```

再看一下两个线程：

```cpp
// This is the server thread that very slowly answers an IPC and exits. Note
// that the pong event needs to be signaled twice.
DWORD WINAPI SlowResponseServer(PVOID param) {
  ServerEvents* events = reinterpret_cast<ServerEvents*>(param);
  DWORD wait_result = 0;
  // 在ping上等待
  wait_result = ::WaitForSingleObject(events->ping, INFINITE);
  // sleep好久。。。
  ::Sleep(kIPCWaitTimeOut1 + kIPCWaitTimeOut2 + 200);
  // 没有处理参数，只是简单的设置信道为Ack
  ::InterlockedExchange(events->state, kAckChannel);
  // 置信pong event
  ::SetEvent(events->pong);
  return wait_result;
}

// This thread's job is to keep the mutex locked.
DWORD WINAPI MainServerThread(PVOID param) {
  ServerEvents* events = reinterpret_cast<ServerEvents*>(param);
  DWORD wait_result = 0;
  // 等待events的mutex，只要这个mutex还拿不到就表示server还在
  wait_result = ::WaitForSingleObject(events->mutex, INFINITE);
  Sleep(kIPCWaitTimeOut1 * 20);
  return wait_result;
}
```