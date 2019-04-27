# Envoy 源码分析－－程序启动过程

>申明：本文的 Envoy 源码分析基于 Envoy1.10.0。

前面几章分析了 [event事件](./envoy_event.md) 和 [底层网络](./envoy_network.md)， 但对创建服务的过程并没有串起来，只是分析了底层的网络公共库。这次我们分析下整个服务的创建过程。

## 初始化

### main 入口

服务启动的总入口 `main` 函数，会先创建 `MainCommon`。

```
int main(int argc, char** argv) {
  ... ...
  std::unique_ptr<Envoy::MainCommon> main_common;

  try {
    main_common = std::make_unique<Envoy::MainCommon>(argc, argv);
  ... ...
}
```

### MainCommon 初始化

`MainCommon` 在构造函数时，会先解析程序的参数，然后再调用 `MainCommonBase`。

```
MainCommon::MainCommon(int argc, const char* const* argv)
    : options_(argc, argv, &MainCommon::hotRestartVersion, spdlog::level::info),
      base_(options_, real_time_system_, default_test_hooks_,prod_component_factory_,
      std::make_unique<Runtime::RandomGeneratorImpl>(),platform_impl_.threadFactory(),
      platform_impl_.fileSystem()) {}
```

`OptionsImpl` 使用开源的 [tclap](https://github.com/eile/tclap) 解析库。`OptionsImpl` 支持很多参数配置，具体的参数配置参考 [operation/cli](https://www.envoyproxy.io/docs/envoy/v1.10.0/operations/cli)。

`MainCommonBase` 会初始化全局的参数，接着调用服务进行初始化。

```
MainCommonBase::MainCommonBase(... ...)
    : options_(options), component_factory_(component_factory), thread_factory_(thread_factory),file_system_(file_system) {
  //全局的或第三方库预先初始化
  Thread::ThreadFactorySingleton::set(&thread_factory_);
  ares_library_init(ARES_LIB_INIT_ALL);
  Event::Libevent::Global::initialize();
  RELEASE_ASSERT(Envoy::Server::validateProtoDescriptors(), "");
  Http::Http2::initializeNghttp2Logging();

  ... ... 

  //初始化统计对象
  stats_store_ = std::make_unique<Stats::ThreadLocalStoreImpl>(options_.statsOptions(),
                                                                 restarter_->statsAllocator());

  //初始化服务
  server_ = std::make_unique<Server::InstanceImpl>(
    options_, time_system, local_address, test_hooks, *restarter_, *stats_store_,
    access_log_lock, component_factory, std::move(random_generator), *tls_, thread_factory_,
    file_system_);
```

### 服务 InstanceImpl 初始化

`InstanceImpl` 主要是初始化 `admin` 管理服务，各个静态 XDS 的加载以及初始化服务。

`InstanceImpl` 在核心函数中先加载配置文件，通过参数 `-c` 配置。

```
void InstanceImpl::initialize(const Options& options,
                              Network::Address::InstanceConstSharedPtr local_address,
                              ComponentFactory& component_factory, TestHooks& hooks) {
  ... ...
  //加载Bootstrap
  InstanceUtil::loadBootstrapConfig(bootstrap_, options, *api_);
  bootstrap_config_update_time_ = time_source_.systemTime();
  ... ...                              
}
```

其配置文件是一个 json 格式，包括以下几项：

+ node：节点标识，会在管理服务器中呈现，用于标识目的（例如，头域中生成相应的字段）。
+ static_resources：指定静态资源配置。
+ dynamic_resources：动态发现服务源配置。
+ cluster_manager：该服务所有的上游群集的群集管理配置。
+ hds_config：服务健康检查配置。
+ flags_path：用于启动文件标志的路径。
+ stats_sinks：统计汇总设置
+ stats_config：配置内部处理统计信息。
+ stats_flush_interval：刷新到统计信息服务的周期时间。出于性能方面的考虑，Envoy锁定计数器，并且只是周期性地刷新计数器和计量器。如果未指定，则默认值为5000毫秒（5秒）。
+ watchdog：看门狗配置。
+ tracing：配置外置的追踪服务程序。如果没有指定，则不会执行追踪。
+ runtime：配置运行时配置分发服务程序。
+ admin： 配置本地管理的HTTP服务。
+ overload_manager：过载管理配置（资源限制）。

各个项的具体配置参考[Bootstrap](https://www.envoyproxy.io/docs/envoy/v1.10.0/api-v2/config/bootstrap/v2/bootstrap.proto)

加载完配置项，接着会启动 `admin` 本地的HTTP服务。

#### 初始化 admin 服务

`admin` 先创建一个 AdminImpl，在构造函数里初始化 URI。

```
AdminImpl::AdminImpl(const std::string& profile_path, Server::Instance& server)
    : server_(server), profile_path_(profile_path),
      ... ...
      handlers_{
          {"/", "Admin home page", MAKE_ADMIN_HANDLER(handlerAdminHome), false, false},
          {"/certs", "print certs on machine", MAKE_ADMIN_HANDLER(handlerCerts), false, false},
          // 导出 cluster  统计信息
          {"/clusters", "upstream cluster status", MAKE_ADMIN_HANDLER(handlerClusters), false,
           false},
          // 导出配置文件
          {"/config_dump", "dump current Envoy configs (experimental)",
           MAKE_ADMIN_HANDLER(handlerConfigDump), false, false},
          // 导出连接统计信息
          {"/contention", "dump current Envoy mutex contention stats (if enabled)",
           MAKE_ADMIN_HANDLER(handlerContention), false, false},
          ... ...
      admin_filter_chain_(std::make_shared<AdminFilterChain>()) {}
```

接着启动一个服务。

```
admin_->startHttpListener(initial_config.admin().accessLogPath(), options.adminAddressPath(),
                              initial_config.admin().address(),
                              stats_store_.createScope("listener.admin."));
```

在启动服务函数内部会创建 TcpListenSocket 和 listen。

```
void AdminImpl::startHttpListener(const std::string& access_log_path,
                                  const std::string& address_out_path,
                                  Network::Address::InstanceConstSharedPtr address,
                                  Stats::ScopePtr&& listener_scope) {
  ... ...
  socket_ = std::make_unique<Network::TcpListenSocket>(address, nullptr, true);
  listener_ = std::make_unique<AdminListener>(*this, std::move(listener_scope));
  ... ...
}
```

初始化 TcpListenSocket 时会在内部创建一个 socket 后再进行 bind。

```
using TcpListenSocket = NetworkListenSocket<NetworkSocketTrait<Address::SocketType::Stream>>;

template <typename T> class NetworkListenSocket : public ListenSocketImpl {
public:
  NetworkListenSocket(const Address::InstanceConstSharedPtr& address,
                      const Network::Socket::OptionsSharedPtr& options, bool bind_to_port)
      // socket 创建
      : ListenSocketImpl(address->socket(T::type), address) {
    RELEASE_ASSERT(io_handle_->fd() != -1, "");

    setPrebindSocketOptions();

    setupSocket(options, bind_to_port);
  }

void ListenSocketImpl::setupSocket(const Network::Socket::OptionsSharedPtr& options, bool bind_to_port) {
  setListenSocketOptions(options);
  //准备进行绑定
  if (bind_to_port) {
    doBind();
  }
}

void ListenSocketImpl::doBind() {
  //绑定
  const Api::SysCallIntResult result = local_address_->bind(io_handle_->fd());
  ... ...
}

```



## 启动

## 
