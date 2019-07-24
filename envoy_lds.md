# Envoy 源码分析－－LDS

>申明：本文的 Envoy 源码分析基于 Envoy1.10.0。

LDS 是 Envoy 用来自动获取 listener 的 API。 Envoy 通过 API 可以增加、修改或删除 listener。

## 初始化

在初始化 bootstrap 配置时，如果有 lds 配置会进行初始化。

```
// Instruct the listener manager to create the LDS provider if needed. This must be done later
// because various items do not yet exist when the listener manager is created.
if (bootstrap_.dynamic_resources().has_lds_config()) {
  listener_manager_->createLdsApi(bootstrap_.dynamic_resources().lds_config());
}
```

在 ListenerManagerImpl 类中通过 ProdListenerComponentFactory 配置类创建 lds。

```
// Server::ListenerManager
void createLdsApi(const envoy::api::v2::core::ConfigSource& lds_config) override {
    ASSERT(lds_api_ == nullptr);
    lds_api_ = factory_.createLdsApi(lds_config);
  }

// Server::ListenerComponentFactory
LdsApiPtr createLdsApi(const envoy::api::v2::core::ConfigSource& lds_config) override {
return std::make_unique<LdsApiImpl>(lds_config, server_.clusterManager(), server_.dispatcher(),
                                    server_.random(), server_.initManager(),
                                    server_.localInfo(), server_.stats(),
                                    server_.listenerManager(), server_.api());
}
```

初始化 lds 时，会创建一个通道。

```
LdsApiImpl::LdsApiImpl(const envoy::api::v2::core::ConfigSource& lds_config,
                   Upstream::ClusterManager& cm, Event::Dispatcher& dispatcher,
                   Runtime::RandomGenerator& random, Init::Manager& init_manager,
                   const LocalInfo::LocalInfo& local_info, Stats::Scope& scope,
                   ListenerManager& lm, Api::Api& api)
: listener_manager_(lm), scope_(scope.createScope("listener_manager.lds.")), cm_(cm),
  init_target_("LDS", [this]() { subscription_->start({}, *this); }) {
  //创建 subscription 对象
  subscription_ = Envoy::Config::SubscriptionFactory::subscriptionFromConfigSource(
    lds_config, local_info, dispatcher, cm, random, *scope_,
    "envoy.api.v2.ListenerDiscoveryService.FetchListeners",
    "envoy.api.v2.ListenerDiscoveryService.StreamListeners",
    Grpc::Common::typeUrl(envoy::api::v2::Listener().GetDescriptor()->full_name()), api);
  Config::Utility::checkLocalInfo("lds", local_info);
  init_manager.add(init_target_);
}
```

## 更新

listener 的更新处理如下：

+ 每个 listener 必须有一个唯一的名称。如果没有提供名称，Envoy 会生成一个 UUID 来作为它的名字。要动态更新 listener，管理服务必须提供一个唯一名称。
+ 当 listener 被添加，在接收流量之前，会先进入 “预热” 阶段。
+ 一旦 listener 被创建，就会保持不变。因此，listener 更新时，会创建一个全新的 listener（同一个侦听套接字）。这个新增加的 listener 同样需要一个 “预热” 过程。
+ 当更新或删除 listener 时，旧的 listener 将被置于 “draining（驱逐）” 状态，和整个服务重新启动时一样。在删除侦听器并关闭任何其余连接之前，侦听器拥有的连接将在一段时间内优雅关闭（如果可能的话）。逐出时间通过 `--drain-time-s` 设置。

有数据下发时，会触发 onConfigUpdate。

```
// the configuration update targets.
callbacks_->onConfigUpdate(resources, version_info);
```

移除 listener。创建一个移除的列表，将现有的 listener 加入，然后去除数据下发的 listener，剩下的就是需要移除的 listener。移除的 listener 会打印 `lds: remove listener '{}'`。

```
  // We build the list of listeners to be removed and remove them before
  // adding new listeners. This allows adding a new listener with the same
  // address as a listener that is to be removed. Do not change the order.
  for (const auto& listener : listener_manager_.listeners()) {
    listeners_to_remove.emplace(listener.get().name(), listener);
  }
  for (const auto& listener : listeners) {
    listeners_to_remove.erase(listener.name());
  }
  for (const auto& listener : listeners_to_remove) {
    if (listener_manager_.removeListener(listener.first)) {
      ENVOY_LOG(info, "lds: remove listener '{}'", listener.first);
    }
  }
```

移除 listener 时，如果是在 ‘预热’ 阶段的 listener 直接删除，如果是在活动的 listener 将其置于驱逐中，过一段时间关闭。

```
  // Destroy a warming listener directly.
  if (existing_warming_listener != warming_listeners_.end()) {
    (*existing_warming_listener)->debugLog("removing warming listener");
    warming_listeners_.erase(existing_warming_listener);
  }

  // If there is an active listener it needs to be moved to draining.
  if (existing_active_listener != active_listeners_.end()) {
    drainListener(std::move(*existing_active_listener));
    active_listeners_.erase(existing_active_listener);
  }


  // ListenerManagerImpl::drainListener
  // 关闭accept，不再接收新连接
  draining_it->listener_->debugLog("draining listener");
  for (const auto& worker : workers_) {
    worker->stopListener(*draining_it->listener_);
  }

  // 等待一段时间
  draining_it->listener_->localDrainManager().startDrainSequence([this, draining_it]() -> void {
    draining_it->listener_->debugLog("removing listener");
    for (const auto& worker : workers_) {
      // 移除 listener
      worker->removeListener(*draining_it->listener_, [this, draining_it]() -> void {
        // 通知主线程（移除是在工作线程移除的，主线程无法知道）
        server_.dispatcher().post([this, draining_it]() -> void {
          if (--draining_it->workers_pending_removal_ == 0) {
            draining_it->listener_->debugLog("listener removal complete");
            draining_listeners_.erase(draining_it);
            stats_.total_listeners_draining_.set(draining_listeners_.size());
          }
        });
      });
    }
  });
```

接下来是对所有的 listener 进行增加或更新。需要增加或更新打印日志`lds: add/update listener '{}'`。

```
  for (const auto& listener : listeners) {
    const std::string& listener_name = listener.name();
    try {
      if (listener_manager_.addOrUpdateListener(listener, version_info, true)) {
        ENVOY_LOG(info, "lds: add/update listener '{}'", listener_name);
      } else {
        ENVOY_LOG(debug, "lds: add/update listener '{}' skipped", listener_name);
      }
    } catch (const EnvoyException& e) {
      exception_msgs.push_back(fmt::format("{}: {}", listener_name, e.what()));
    }
  }
```

增加 listener 时，如果没有名称，Envoy 生成一个 UUID。更新 listener 如果没有名称，Envoy 生成 UUID，无法通过名称进行关联，不带名称进行更新，只会变成新增。

```
  std::string name;
  if (!config.name().empty()) {
    name = config.name();
  } else {
    name = server_.random().uuid();
  }
```

