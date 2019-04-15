# Envoy 源码分析－－network

>申明：本文的 Envoy 源码分析基于 Envoy1.10.0。

`Envoy` 的服务是通用服务，因此它需要支持 `TCP` 和 `UDP`，同时还需支持 `IPV4` 和 `IPV6` 两种网络协议，所以网络模块有点复杂。本次分析的网络模块是底层的模块，没有一整个服务的启动流程，有的地方可能还串不起来。现在先来看下UML类图：

![envoy_network](./images/envoy_network.png)

类图看上去略显复杂，主要分为4块：`addres`，`socket`，`listen` 和 `connection`。

+ `address` 是地址相关的，主要包括 `IPV4`，`IPV6`，`PIPE`，`DNS` 和 `cidr`。
+ `socket` 是 `socket` 相关的操作，主要包括 `ListenSocket`，`ConnectionSocket`，`TransportSocket` 以及 `option` 。
+ `listen` 是网络监听操作，包括 `TCP` 监听和 `UDP` 监听。
+ `connection` 是连接相关操作。关于 L3/4 过滤的这次暂时不分析，后续再讲。


## address

`InstanceBase` 继承自 `Instance` 是所有地址类型的基类。`Ipv4Instance`，`Ipv6Instance` 和 `PipeInstance` 三个地址类都是继承 `InstanceBase`。 `DNS`解析类使用 `c-ares` 库，`DnsResolverImpl` 只是对 `c-ares` 的进一步封装。 `CidrRange` 是对 [cidr](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 操作相关。

系统操作返回的值和错误信息封装成一个公用的结构体。具体如下：

```
template <typename T> struct SysCallResult {
  //系统返回值
  T rc_;
  //系统返回的错误信息
  int errno_;
};
```

### Instance

`Ipv4Instance`，`Ipv6Instance` 和 `PipeInstance` 三个地址类都是继承 `InstanceBase`。它们的实现基本都差不多，`socket()`、`bind()` 和 `connect()` 这三个基础操作都属于它们的成员。现在我们主要来看下 `Ipv4Instance` 几个主要的操作（其它两个类类似就不再分析）。

`Ipv4Instance` 的类里有个私有结构体 `IpHelper` 。这结构体封装着 `IPV4` 地址的具体内容，比如端口，版本等

```
struct IpHelper : public Ip {
  const std::string& addressAsString() const override { return friendly_address_; }
  bool isAnyAddress() const override { return ipv4_.address_.sin_addr.s_addr == INADDR_ANY; }
  bool isUnicastAddress() const override {
  return !isAnyAddress() && (ipv4_.address_.sin_addr.s_addr != INADDR_BROADCAST) &&
             // inlined IN_MULTICAST() to avoid byte swapping
             !((ipv4_.address_.sin_addr.s_addr & htonl(0xf0000000)) == htonl(0xe0000000));
  }
  const Ipv4* ipv4() const override { return &ipv4_; }
  const Ipv6* ipv6() const override { return nullptr; }
  uint32_t port() const override { return ntohs(ipv4_.address_.sin_port); }
  IpVersion version() const override { return IpVersion::v4; }

  Ipv4Helper ipv4_;
  std::string friendly_address_;
};
```

`bind()`，`socket()` 和 `connect()` 基本都是直接调的底层函数。

```
Api::SysCallIntResult Ipv6Instance::bind(int fd) const {
  const int rc = ::bind(fd, reinterpret_cast<const sockaddr*>(&ip_.ipv6_.address_),
                        sizeof(ip_.ipv6_.address_));
  return {rc, errno};
}

Api::SysCallIntResult Ipv6Instance::connect(int fd) const {
  const int rc = ::connect(fd, reinterpret_cast<const sockaddr*>(&ip_.ipv6_.address_),
                           sizeof(ip_.ipv6_.address_));
  return {rc, errno};
}
```

### DNS

`DNS` 使用 [c-ares](https://c-ares.haxx.se/) 作为底层库。 `c-ares` 是个 c 实现的异步 DNS 解析库，很多知名软件（curl，Nodejs，gevent 等）都使用了该库。

`c-ares` 在构造函数内初始化库，初始化上下文，然后设置 DNS 服务器。

```
DnsResolverImpl::DnsResolverImpl(
    Event::Dispatcher& dispatcher,
    const std::vector<Network::Address::InstanceConstSharedPtr>& resolvers)
        : dispatcher_(dispatcher),
      timer_(dispatcher.createTimer([this] { onEventCallback(ARES_SOCKET_BAD, 0); })) {
  //初始化库
  ares_library_init(ARES_LIB_INIT_ALL);
  ares_options options;
  //初始化上下文
  initializeChannel(&options, 0);
  ... ...
  const std::string resolvers_csv = StringUtil::join(resolver_addrs, ",");
  //设置 DNS 服务器
  int result = ares_set_servers_ports_csv(channel_, resolvers_csv.c_str());
}
```

使用时直接 `resolve()` 结果返回在 callback 里。

```
ActiveDnsQuery* DnsResolverImpl::resolve(const std::string& dns_name,
                                         DnsLookupFamily dns_lookup_family, ResolveCb callback) {
  ... ...
  if (dns_lookup_family == DnsLookupFamily::V4Only) {
    pending_resolution->getHostByName(AF_INET);
  } else {
    pending_resolution->getHostByName(AF_INET6);
  }
  ... ...
}

void DnsResolverImpl::PendingResolution::getHostByName(int family) {
  ares_gethostbyname(channel_, dns_name_.c_str(), family,
                     [](void* arg, int status, int timeouts, hostent* hostent) {
                       static_cast<PendingResolution*>(arg)->onAresHostCallback(status, timeouts, hostent);
                     },
                     this);
}

void DnsResolverImpl::PendingResolution::onAresHostCallback(int status, int timeouts, hostent* hostent) {

  ... ...
  //解析内容加入address_list
  std::list<Address::InstanceConstSharedPtr> address_list;
  if (status == ARES_SUCCESS) {
    if (hostent->h_addrtype == AF_INET) {
      for (int i = 0; hostent->h_addr_list[i] != nullptr; ++i) {
        ASSERT(hostent->h_length == sizeof(in_addr));
        sockaddr_in address;
        memset(&address, 0, sizeof(address));
        address.sin_family = AF_INET;
        address.sin_port = 0;
        address.sin_addr = *reinterpret_cast<in_addr*>(hostent->h_addr_list[i]);
        address_list.emplace_back(new Address::Ipv4Instance(&address));
      }
      ... ...
  }

  if (completed_) {
    if (!cancelled_) {
      try {
        //调用回调
        callback_(std::move(address_list));
      } catch (const EnvoyException& e) {
      ... ...
}
```

### cidr

`cidr` 的定义是形如 192.168.0.1/24 的 IP 段。想知道具体的定义和 IP 段 可看 [cidr](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)。

`CidrRange` 将 `cidr` 拆分成两字段地址和长度。下面是判断地址是否属于这个 IP 段。

```
bool CidrRange::isInRange(const Instance& address) const {
  ... ...
  //长度为0，全匹配（length_初始值为-1）
  if (length_ == 0) {
    return true;
  }

  switch (address.ip()->version()) {
  case IpVersion::v4:
    if (ntohl(address.ip()->ipv4()->address()) >> (32 - length_) ==
        ntohl(address_->ip()->ipv4()->address()) >> (32 - length_)) {
      return true;
    }
    break;
  case IpVersion::v6:
    if ((Utility::Ip6ntohl(address_->ip()->ipv6()->address()) >> (128 - length_)) ==
        (Utility::Ip6ntohl(address.ip()->ipv6()->address()) >> (128 - length_))) {
      return true;
    }
    break;
  }
  return false;
}
```


## socket

我们都知道，创建 TCP 服务时，监听的 fd 和连接的 fd 是不一样的，因此 `socket` 分为 `ListenSocket` 和 `ConnectionSocket`。`socket` 里有很多的配置（比如读超时，写超时等）都是调用`setsockopt`，所有需要一个 `Option` 来进行统一的封装。

### Option

`Option` 是对 `setsockopt` 这个函数操作的封装。封装后再用智能指针的方式进行操作。

```
typedef std::shared_ptr<const Option> OptionConstSharedPtr;
typedef std::vector<OptionConstSharedPtr> Options;
typedef std::shared_ptr<Options> OptionsSharedPtr;
```

`Option` 在全部设置完后，在 `applyOptions`后，最终还是调用 `setsockopt`。

```
static bool applyOptions(const OptionsSharedPtr& options, Socket& socket,
                       envoy::api::v2::core::SocketOption::SocketState state) {
  if (options == nullptr) {
    return true;
  }
  for (const auto& option : *options) {
    //对所有的option 进行设置
    if (!option->setOption(socket, state)) {
      return false;
    }
  }
  return true;
}

bool SocketOptionImpl::setOption(Socket& socket,
                                 envoy::api::v2::core::SocketOption::SocketState state) const {
  if (in_state_ == state) {
    //调用成员函数 setSocketOption
    const Api::SysCallIntResult result = SocketOptionImpl::setSocketOption(socket, optname_, value_);
  ... ...
  return true;
}

Api::SysCallIntResult SocketOptionImpl::setSocketOption(Socket& socket, Network::SocketOptionName optname, const absl::string_view value) {
  ... ...
  //最终调用系统函数setsockopt
  return os_syscalls.setsockopt(socket.ioHandle().fd(), optname.value().first,
                                optname.value().second, value.data(), value.size());
}
```

### Socket

`Socket` 提供基本的 socket 操作。主要是 'Option' 操作（上面已分析过）和地址操作。代码比较简单。

```
//设置和获取本地地址
const Address::InstanceConstSharedPtr& localAddress() const override { return local_address_; }
void setLocalAddress(const Address::InstanceConstSharedPtr& local_address) override {
  local_address_ = local_address;
}
```

### ListenSocket

`ListenSocket` 是对监听 fd 的封装，继承自 `Socket`。主要操作自然就是 `bind()`。bind 调用自地址类的 bind() 函数（看上面的 address）。

```
void ListenSocketImpl::doBind() {
  // 地址和handle 继承自socket。调用地址类的 bind。
  const Api::SysCallIntResult result = local_address_->bind(io_handle_->fd());
  if (result.rc_ == -1) {
    close();
    throw SocketBindException(
        fmt::format("cannot bind '{}': {}", local_address_->asString(), strerror(result.errno_)),
        result.errno_);
  }
  if (local_address_->type() == Address::Type::Ip && local_address_->ip()->port() == 0) {
    // If the port we bind is zero, then the OS will pick a free port for us (assuming there are
    // any), and we need to find out the port number that the OS picked.
    local_address_ = Address::addressFromFd(io_handle_->fd());
  }

```

### ConnectionSocket

`ConnectionSocket` 是对连接 fd 的封装，除了 `Socket` 的基本操作外，还增加对远程地址和协议的设置。

```
//设置和获取远程地址
const Address::InstanceConstSharedPtr& remoteAddress() const override { return remote_address_; }
void setRemoteAddress(const Address::InstanceConstSharedPtr& remote_address) override {
  remote_address_ = remote_address;
}

//协议相关
void setDetectedTransportProtocol(absl::string_view protocol) override {
 transport_protocol_ = std::string(protocol);
}
absl::string_view detectedTransportProtocol() const override { return transport_protocol_; }
```

### TransportSocket

`TransportSocket` 是一个实际读/写的传输套接字。它可以对数据进行一些转换（比如TLS，TCP代理等）。 `TransportSocket` 提供了多个接口。

`failureReason()` 返回最后的一个错误，没错误返回空值。

`canFlushClose()` socket 是否能刷新和关闭。

`closeSocket()` 关闭 socket。

`doRead()` 读取数据。

`doWrite()` 写数据。

`onConnected()` transport 连接时调用此函数。

`Ssl::ConnectionInfo* ssl()` Ssl连接数据。


## listen

`listen` 是对监听操作相关的类，分为 `TcpListen` 和 `UdpListen`。 Listen 抽象类只提供两个接口 `disable` 和 `enable`。 `disable` 关闭接受新连接，`enable`开启接受新连接。 

`ListenerImpl` 实现那两接口的同时，由于它是 TCP 的监听必然就有 listen 和 accept 操作。在构造函数时，调用 setupServerSocket 创造 listen，启用回调

```
void ListenerImpl::
setupServerSocket(Event::DispatcherImpl& dispatcher, Socket& socket) {
  //创建监听，完成后回调 listenCallback
  listener_.reset(
      evconnlistener_new(&dispatcher.base(), listenCallback, this, 0, -1, socket.ioHandle().fd()));
  ... ...
  //失败回调errorCallback
  evconnlistener_set_error_cb(listener_.get(), errorCallback);
}
```

监听完成后，调用 listenCallback。listenCallback 用回调函数调用 onAccept 接收连接。

```

void ListenerImpl::listenCallback(evconnlistener*, evutil_socket_t fd, sockaddr* remote_addr, int remote_addr_len, void* arg) {
  ListenerImpl* listener = static_cast<ListenerImpl*>(arg);

  IoHandlePtr io_handle = std::make_unique<IoSocketHandleImpl>(fd);

  // 获取本地地址
  const Address::InstanceConstSharedPtr& local_address =
      listener->local_address_ ? listener->local_address_
                               : listener->getLocalAddress(io_handle->fd());
  // 获取远程地址
  const Address::InstanceConstSharedPtr& remote_address =
      (remote_addr->sa_family == AF_UNIX)
          ? Address::peerAddressFromFd(io_handle->fd())
          : Address::addressFromSockAddr(*reinterpret_cast<const sockaddr_storage*>(remote_addr),
                                         remote_addr_len,
                                         local_address->ip()->version() == Address::IpVersion::v6);
  //调用 onAccept，
  listener->cb_.onAccept(
      std::make_unique<AcceptedSocketImpl>(std::move(io_handle), local_address, remote_address),
      listener->hand_off_restored_destination_connections_);
}
```


## connection

`connection` 是连接相关的操作，客户端和服务端的连接都属于这个类。 `Connection` 是针对原始连接的一个抽象，继承自 `DeferredDeletable` 和 `FilterManager`。关于 `DeferredDeletable` 延迟析构请看 [Envoy 源码分析－－event](https://www.cnblogs.com/mathli/p/10674391.html)，`FilterManager` 以后讨论。

### ConnectionImpl

`ConnectionImpl` 是 `Connection`，`BufferSource` 和 `TransportSocketCallbacks` 三个抽象类的实现类。`Connection` 是连接操作相关的类，`BufferSource` 是获得 StreamBuffer 的抽象类（包括读和写），`TransportSocketCallbacks` 是传输套接字实例与连接进行通信的回调。

每个 `ConnectionImpl` 实例都有一个唯一的全局ID。在构造时赋值。

```
std::atomic<uint64_t> ConnectionImpl::next_global_id_;

ConnectionImpl::ConnectionImpl(Event::Dispatcher& dispatcher, ConnectionSocketPtr&& socket, TransportSocketPtr&& transport_socket, bool connected) : id_(next_global_id_++) {
}
```
 
`ConnectionImpl` 事件由 `dispatcher_` 创建。在构造函数时创建事件。
Event 使用边缘触发，减少内核通知，提高效率（水平触发和边缘触发区别大家自己查阅相关文档）。同时写入读写事件。当有读写事件时，会触发回调 `onFileEvent`。

```
ConnectionImpl::ConnectionImpl(Event::Dispatcher& dispatcher, ConnectionSocketPtr&& socket,TransportSocketPtr&& transport_socket, bool connected) {
  ... ...
  file_event_ = dispatcher_.createFileEvent(
      ioHandle().fd(), [this](uint32_t events) -> void { onFileEvent(events); },
      Event::FileTriggerType::Edge, Event::FileReadyType::Read | Event::FileReadyType::Write);
  ... ...
}
```

`onFileEvent` 在收到事件后，对不同的事件进行不同的处理。

```
void ConnectionImpl::onFileEvent(uint32_t events) {
  ... ...
  // 写事件
  if (events & Event::FileReadyType::Write) {
    onWriteReady();
  }

  // 读事件
  if (ioHandle().isOpen() && (events & Event::FileReadyType::Read)) {
    onReadReady();
  }
}
```

对于读事件，在连接调用 `readDisable` 后，如果是 enable 会触发读事件。

```
void ConnectionImpl::readDisable(bool disable) {
    ... ...
    read_enabled_ = true;
    file_event_->setEnabled(Event::FileReadyType::Read | Event::FileReadyType::Write);
    if (read_buffer_.length() > 0) {
      file_event_->activate(Event::FileReadyType::Read);
    }
}
```

读事件调用 `onReadReady`，`onReadReady` 先从 buffer中读取数据，同时更新统计数据。对返回的结果进行分析，已关闭直接关闭。正常读到数据，判断是否有数据，有数据会调用 `onRead`, `onRead` 内会调用 ReadFilter 进行下一步处理（L3/4过滤下次分析）。 

```
void ConnectionImpl::onReadReady() {
  ... ...
  IoResult result = transport_socket_->doRead(read_buffer_);
  uint64_t new_buffer_size = read_buffer_.length();
  updateReadBufferStats(result.bytes_processed_, new_buffer_size);

  if ((!enable_half_close_ && result.end_stream_read_)) {
    result.end_stream_read_ = false;
    result.action_ = PostIoAction::Close;
  }

  read_end_stream_ |= result.end_stream_read_;
  //有读到数据
  if (result.bytes_processed_ != 0 || result.end_stream_read_) 
    onRead(new_buffer_size);
  }

  // 关闭连接
  if (result.action_ == PostIoAction::Close || bothSidesHalfClosed()) {
    ENVOY_CONN_LOG(debug, "remote close", *this);
    closeSocket(ConnectionEvent::RemoteClose);
  }
}
```

对于写事件，在连接写入数据时，会将数据先进行过滤，然后写入写缓冲。之后调用写事件触发 `onFileEvent`

```
void ConnectionImpl::write(Buffer::Instance& data, bool end_stream) {
  ... ...
  // WriteFilter过滤
  current_write_buffer_ = &data;
  current_write_end_stream_ = end_stream;
  FilterStatus status = filter_manager_.onWrite();
  current_write_buffer_ = nullptr;

  if (FilterStatus::StopIteration == status) {
    return;
  }

  write_end_stream_ = end_stream;
  if (data.length() > 0 || end_stream) {
    // 写入缓冲
    write_buffer_->move(data);
    if (!connecting_) {
      //触发写事件
      file_event_->activate(Event::FileReadyType::Write);
    }
  }
}
```

在写入事件后会调用 `onWriteReady`。 `onWriteReady` 先判断是否已连接，未连接会调用 `connect` 连接事件。连接成功后发送数据并统计信息，连接失败关闭 socket。

```
void ConnectionImpl::onWriteReady() {
  ... ...
  if (connecting_) {
    ... ...
    if (error == 0) {
      connecting_ = false;
      //socket 未连接，调用connect。
      transport_socket_->onConnected();
     ... ...

  // 发送数据
  IoResult result = transport_socket_->doWrite(*write_buffer_, write_end_stream_);
  uint64_t new_buffer_size = write_buffer_->length();
  //更新统计信息
  updateWriteBufferStats(result.bytes_processed_, new_buffer_size);
  ... ...
}
```

### ClientConnectionImpl

`ClientConnectionImpl` 是客户端的连接，其继承自 `ConnectionImpl` 和 `ClientConnection`。`ClientConnectionImpl` 只是在 `Connection` 的基础上只增加了一个 connect 的接口。

connect 函数内最主要做的就是调用 connect() 连接。

```
void ClientConnectionImpl::connect() {
  // 连接服务器
  const Api::SysCallIntResult result = socket_->remoteAddress()->connect(ioHandle().fd());
  if (result.rc_ == 0) {
    // write will become ready.
    ASSERT(connecting_);
  } else {
  ... ...
}
```

