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




## 启动

## 
