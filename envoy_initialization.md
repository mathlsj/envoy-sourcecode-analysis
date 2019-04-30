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

配置文件是一个 json 格式，包括以下几项：

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

各个项的具体作用和配置参考[Bootstrap](https://www.envoyproxy.io/docs/envoy/v1.10.0/api-v2/config/bootstrap/v2/bootstrap.proto)

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

在启动服务函数内部会创建 TcpListenSocket 和 AdminListener。

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

AdminListener 构造函数内只是参数的初始化。

```
AdminListener(AdminImpl& parent, Stats::ScopePtr&& listener_scope)
  : parent_(parent), name_("admin"), scope_(std::move(listener_scope)),
    stats_(Http::ConnectionManagerImpl::generateListenerStats("http.admin.", *scope_)) {}
```

做完 socket ，bind 后面就是进行 listen 处理。将 AdminListener 通过 handler 加入监听队列。handler 是在 `InstanceImpl` 的构造函数内初始化的。

```
InstanceImpl::InstanceImpl(... ...）
  : handler_(new ConnectionHandlerImpl(ENVOY_LOGGER(), *dispatcher_)),
  ... ...{
  }

void InstanceImpl::initialize(... ...)
{
  ... ...
  //将 AdminListener 加入 ConnectionHandler
  if (initial_config.admin().address()) {
    admin_->addListenerToHandler(handler_.get());
  }
  ... ...
}

void AdminImpl::addListenerToHandler(Network::ConnectionHandler* handler) {
  // 这里的 listener_ 就是上面生成的 AdminListener
  if (listener_) {
    handler->addListener(*listener_);
  }
}
```

在 addListener 内会新建一个 ActiveListener 内部类，先置为 disable 状态。

```
void ConnectionHandlerImpl::addListener(Network::ListenerConfig& config) {
  ActiveListenerPtr l(new ActiveListener(*this, config));
  if (disable_listeners_) {
    l->listener_->disable();
  }
  listeners_.emplace_back(config.socket().localAddress(), std::move(l));
}
```

在 ActiveListener 构造函数内创建 listen，里面dispatcher 会创建回调。等有新连接到来时，会回调 onAccept.

```
ConnectionHandlerImpl::ActiveListener::ActiveListener(ConnectionHandlerImpl& parent,Network::ListenerConfig& config)
    : ActiveListener(
          parent,
          // 创建listen
          parent.dispatcher_.createListener(config.socket(), *this, config.bindToPort(),
                                            config.handOffRestoredDestinationConnections()),
          config) {}

Network::ListenerPtr
DispatcherImpl::createListener(Network::Socket& socket, Network::ListenerCallbacks& cb, bool bind_to_port, bool hand_off_restored_destination_connections) {
  ASSERT(isThreadSafe());
  return Network::ListenerPtr{new Network::ListenerImpl(*this, socket, cb, bind_to_port,
                                                        hand_off_restored_destination_connections)};
}

void ListenerImpl::setupServerSocket(Event::DispatcherImpl& dispatcher, Socket& socket) {
  listener_.reset(
      //创建 evconnlistener_new ,有连接回调listenCallback
      evconnlistener_new(&dispatcher.base(), listenCallback, this, 0, -1, socket.ioHandle().fd()));
  ... ...
}

void ListenerImpl::listenCallback(evconnlistener*, evutil_socket_t fd, sockaddr* remote_addr,int remote_addr_len, void* arg) {
  ... ...

  //回调ActiveListener的onAccept
  listener->cb_.onAccept(
    std::make_unique<AcceptedSocketImpl>(std::move(io_handle), local_address, remote_address),
    listener->hand_off_restored_destination_connections_);
}
```

onAccept 对 Listern 过滤后，创建新连接。

```
void ConnectionHandlerImpl::ActiveListener::onAccept(）
{
  ... ...
  active_socket->continueFilterChain(true);
  ... ...
}

void ConnectionHandlerImpl::ActiveSocket::continueFilterChain(bool success) {
  ... ...
   listener_.newConnection(std::move(socket_));
  ... ...
}

void ConnectionHandlerImpl::ActiveListener::onNewConnection() {
  ... ...
  if (new_connection->state() != Network::Connection::State::Closed) {
    ActiveConnectionPtr active_connection(
        new ActiveConnection(*this, std::move(new_connection), parent_.dispatcher_.timeSource()));
    active_connection->moveIntoList(std::move(active_connection), connections_);
    parent_.num_connections_++;
  }
  ... ...
}
```

这样，新连接就建立起来。

#### 配置文件 XDS 初始化 

初始化 Bootstrap 的 XDS 时，先初始化 static sercret，先初始化 cluster，接着初始化 listeners。

```
void MainImpl::initialize(... ...) {
  //初始化secrets
  const auto& secrets = bootstrap.static_resources().secrets();
  for (ssize_t i = 0; i < secrets.size(); i++) {
    ENVOY_LOG(debug, "static secret #{}: {}", i, secrets[i].name());
    server.secretManager().addStaticSecret(secrets[i]);
  }

  //初始化 cluster
  bootstrap.static_resources().clusters().size());
  cluster_manager_ = cluster_manager_factory.clusterManagerFromProto(bootstrap);

  // 初始化listeners
  const auto& listeners = bootstrap.static_resources().listeners();
  for (ssize_t i = 0; i < listeners.size(); i++) {
    ENVOY_LOG(debug, "listener #{}:", i);
    server.listenerManager().addOrUpdateListener(listeners[i], "", false);
  }
```

初始化 cluster 会分两阶段初始化。先初始化非 EDS 部分，再初始化 EDS 部分。分两个阶段的初始化是因为在 v2 配置中每个 EDS 集群单独设置订阅。此订阅是 API 源时  群集将依赖于非 EDS 群集，因此必须首先初始化非 EDS 群集。cluster 的类型有 5个类型：

```
enum DiscoveryType {
    // Refer to the :ref:`static discovery 
    STATIC = 0;

    // Refer to the :ref:`strict DNS discovery
    STRICT_DNS = 1;

    // Refer to the :ref:`logical DNS discovery
    LOGICAL_DNS = 2;

    // Refer to the :ref:`service discovery 
    EDS = 3;

    // Refer to the :ref:`original destination discovery
    ORIGINAL_DST = 4;
  }
```

cluster 初始化顺序：
非 EDS 部分 -> ADS -> EDS -> CDS

```
ClusterManagerImpl::ClusterManagerImpl(... ...) {
  ... ...
  for (const auto& cluster : bootstrap.static_resources().clusters()) {
    // 第一次初始化非 EDS 部分
    if (cluster.type() != envoy::api::v2::Cluster::EDS) {
      loadCluster(cluster, "", false, active_clusters_);
    }
  }

  // 初始化 ADS
  if (bootstrap.dynamic_resources().has_ads_config()) {
    ads_mux_ = std::make_unique<Config::GrpcMuxImpl>(
        local_info,
        Config::Utility::factoryForGrpcApiConfigSource(
            *async_client_manager_, bootstrap.dynamic_resources().ads_config(), stats)
            ->create(),
        main_thread_dispatcher,
        *Protobuf::DescriptorPool::generated_pool()->FindMethodByName(
            "envoy.service.discovery.v2.AggregatedDiscoveryService.StreamAggregatedResources"),
        random_, stats_,
        Envoy::Config::Utility::parseRateLimitSettings(bootstrap.dynamic_resources().ads_config()));
  } else {
    ads_mux_ = std::make_unique<Config::NullGrpcMuxImpl>();
  }

  for (const auto& cluster : bootstrap.static_resources().clusters()) {
    // 初始化 EDS
    if (cluster.type() == envoy::api::v2::Cluster::EDS) {
      loadCluster(cluster, "", false, active_clusters_);
    }
  }

  ... ...
  //初始化 CDS
  if (bootstrap.dynamic_resources().has_cds_config()) {
    cds_api_ = factory_.createCds(bootstrap.dynamic_resources().cds_config(), *this);
    init_helper_.setCds(cds_api_.get());
  } else {
    init_helper_.setCds(nullptr);
  }

}
```

上面都初始化完成后，再初始化 lds，最后再初始化 hds。

```
// 初始化lds
if (bootstrap_.dynamic_resources().has_lds_config()) {
  listener_manager_->createLdsApi(bootstrap_.dynamic_resources().lds_config());
}

//初始化hds
if (bootstrap_.has_hds_config()) {
  const auto& hds_config = bootstrap_.hds_config();
  async_client_manager_ = std::make_unique<Grpc::AsyncClientManagerImpl>(
      *config_.clusterManager(), thread_local_, time_source_, *api_);
  ... ...
}
```

#### 初始化 ListenerManager 

ListenerManager 的初始化只是事先创建 worker。

```
ListenerManagerImpl::ListenerManagerImpl(... ...) {
  ... ...
  // 创建worker子线程
  for (uint32_t i = 0; i < server.options().concurrency(); i++) {
    workers_.emplace_back(worker_factory.createWorker(server.overloadManager()));
  }
}

WorkerPtr ProdWorkerFactory::createWorker(OverloadManager& overload_manager) {
  // 新建子线程，每个线种一个dispatchr
  Event::DispatcherPtr dispatcher(api_.allocateDispatcher());
  return WorkerPtr{new WorkerImpl(
      tls_, hooks_, std::move(dispatcher),
      Network::ConnectionHandlerPtr{new ConnectionHandlerImpl(ENVOY_LOGGER(), *dispatcher)},
      overload_manager, api_)};
}

WorkerImpl::WorkerImpl(... ...)
    : tls_(tls), hooks_(hooks), dispatcher_(std::move(dispatcher)), handler_(std::move(handler)), api_(api) {
  tls_.registerThread(*dispatcher_, false);
  overload_manager.registerForAction(
      OverloadActionNames::get().StopAcceptingConnections, *dispatcher_,
      [this](OverloadActionState state) { stopAcceptingConnectionsCb(state); });
}
```

## 启动

### main 启动入口

main 函数调用 main_common

```
int main(int argc, char** argv) {
  ... ...
  return main_common->run() ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

main_common 进一步调用 InstanceImpl

```
bool MainCommonBase::run() {
  switch (options_.mode()) {
  case Server::Mode::Serve:
    server_->run();
    return true;
  ... ...
}
```

InstanceImpl 启用 loop 循环。

```
void InstanceImpl::run() {
  auto run_helper = RunHelper(*this, options_, *dispatcher_, clusterManager(), access_log_manager_,
                              init_manager_, overloadManager(), [this] { startWorkers(); });

  auto watchdog = guard_dog_->createWatchDog(api_->threadFactory().currentThreadId());
  watchdog->startWatchdog(*dispatcher_);
  dispatcher_->post([this] { notifyCallbacksForStage(Stage::Startup); });
  dispatcher_->run(Event::Dispatcher::RunType::Block);
  ENVOY_LOG(info, "main dispatch loop exited");
  guard_dog_->stopWatching(watchdog);
  watchdog.reset();

  terminate();
}
```

### 服务启动流程

本地 HTTP 管理服务的启动流程上面已经分析过，现在讨论本地服务的启动流程（XDS 下发的暂不讨论）。

在 cluster 初始化的时候，加入 listener。

```
void MainImpl::initialize(... ...) {
  ... ...
  // 初始化listeners
  const auto& listeners = bootstrap.static_resources().listeners();
  for (ssize_t i = 0; i < listeners.size(); i++) {
    ENVOY_LOG(debug, "listener #{}:", i);
    server.listenerManager().addOrUpdateListener(listeners[i], "", false);
  }
}
```

addOrUpdateListener 创建 ListenerImpl，ListenerImpl 做 bind 操作。

```
  // 创建ListenerImpl
  ListenerImplPtr new_listener(
      new ListenerImpl(config, version_info, *this, name, modifiable, workers_started_, hash));
  ListenerImpl& new_listener_ref = *new_listener;
  ... ...

  //bind 地址将 socket 关联ListenerImpl
  new_listener->setSocket(draining_listener_socket
                                ? draining_listener_socket
                                : factory_.createListenSocket(new_listener->address(),
                                                              new_listener->socketType(),
                                                              new_listener->listenSocketOptions(),
                                                              new_listener->bindToPort()));

Network::SocketSharedPtr ProdListenerComponentFactory::createListenSocket(... ...) {
  ... ...
  // 调用 UdsListenSocket 做 bind() 操作。
  if (io_handle->isOpen()) {
      return std::make_shared<Network::UdsListenSocket>(std::move(io_handle), address);
  }
  return std::make_shared<Network::UdsListenSocket>(address);
}

// 最终调用系统bind()操作
void ListenSocketImpl::doBind() {
  const Api::SysCallIntResult result = local_address_->bind(io_handle_->fd());
  ... ...
}
```

在 InstanceImpl 启动时，调用 RunHelper。RunHelper 则启动 startWorkers。startWorker 将初始化得到的 listeners 加入到 work 中。

```
void ListenerManagerImpl::startWorkers(GuardDog& guard_dog) {
  workers_started_ = true;
  for (const auto& worker : workers_) {
    ASSERT(warming_listeners_.empty());
    for (const auto& listener : active_listeners_) {
      addListenerToWorker(*worker, *listener);
    }
    worker->start(guard_dog);
  }
}
```

work 将 linsteners 关联到 connectioHandler。

```
void ListenerManagerImpl::addListenerToWorker(Worker& worker, ListenerImpl& listener) {
  worker.addListener(listener, [this, &listener](bool success) -> void {
  ... ...
}

void WorkerImpl::addListener(Network::ListenerConfig& listener, AddListenerCompletion completion) {
  dispatcher_->post([this, &listener, completion]() -> void {
    try {
      // 关联到connectioHandler。
      handler_->addListener(listener);
      hooks_.onWorkerListenerAdded();
      completion(true);
    } catch (const Network::CreateListenerException& e) {
      completion(false);
    }
  });
}
```

connectioHandler 在 work 初始化时创建。

```
ListenerManagerImpl::ListenerManagerImpl(Instance& server,
                                         ListenerComponentFactory& listener_factory,
                                         WorkerFactory& worker_factory)
    : server_(server), factory_(listener_factory), stats_(generateStats(server.stats())),
      config_tracker_entry_(server.admin().getConfigTracker().add(
          "listeners", [this] { return dumpListenerConfigs(); })) {
  for (uint32_t i = 0; i < server.options().concurrency(); i++) {
    // 初始化worker
    workers_.emplace_back(worker_factory.createWorker(server.overloadManager()));
  }
}

WorkerPtr ProdWorkerFactory::createWorker(OverloadManager& overload_manager) {
  Event::DispatcherPtr dispatcher(api_.allocateDispatcher());
  return WorkerPtr{new WorkerImpl(
      tls_, hooks_, std::move(dispatcher),
      //创建connectioHandler
      Network::ConnectionHandlerPtr{new ConnectionHandlerImpl(ENVOY_LOGGER(), *dispatcher)},
      overload_manager, api_)};
}
```

将 linsteners 关联到 connectioHandler 后，后面的 listen()，accept() 和创建连接过程和 `admin` 的 HTTP 启动流程是一样的。

### LDS 服务启动流程

整个服务的启动流程基本就完成了，后面有新加服务的启动流程和上面的服务启动流程一样，调用 addOrUpdateListener。在 addOrUpdateListener 内判断服务是否已启动，如果已启动调用 ManagerImpl 等待初始化。

```
void ListenerImpl::initialize() {
  last_updated_ = timeSource().systemTime();
  if (workers_started_) {
    //ManagerImpl
    dynamic_init_manager_.initialize(*init_watcher_);
  }
}

void ManagerImpl::initialize(const Watcher& watcher) {
  ... ...
    for (const auto& target_handle : target_handles_) {
      // 等待 target_handle 初始化完成。
      if (!target_handle->initialize(watcher_)) {
        onTargetReady();
      }
    }
}
```

初始化完成后，调用函数指针。函数指针在初始化WatcherImpl传入。

```
void ManagerImpl::onTargetReady() {
  if (--count_ == 0) {
    // 初始化完成
    ready();
  }
}

void ManagerImpl::ready() {
  state_ = State::Initialized;
  watcher_handle_->ready();
}

bool WatcherHandleImpl::ready() const {
    //调用函数指针
    (*locked_fn)();
}

ListenerImpl::ListenerImpl(... ...)
  : ... ...
   // 初始化watch
   init_watcher_(std::make_unique<Init::WatcherImpl>(
          "ListenerImpl", [this] { parent_.onListenerWarmed(*this); })){}
```

在 onListenerWarmed 内将 listener 加入 work。后面流程和 服务启动流程一样，不再分析。

```
void ListenerManagerImpl::onListenerWarmed(ListenerImpl& listener) {
  for (const auto& worker : workers_) {
    addListenerToWorker(*worker, listener);
  }
```

## 最后

整个服务的初始化和启动流程就完成了。服务的启动有3个类型 ： 本地 HTTP 服务管理服务、本地配置文件的服务和xDS下发的服务。本章节只分析了服务的启动流程，连接成功的后继处理，以后分析。