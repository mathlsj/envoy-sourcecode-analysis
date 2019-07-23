# Envoy 源码分析－－LDS

>申明：本文的 Envoy 源码分析基于 Envoy1.10.0。

LDS 是 Envoy 用来自动获取 listener 的 API。 Envoy 通过 API 可以增加、修改或删除 listener。

listener 的更新处理如下：

+ 每个 listener 必须有一个唯一的名字。如果没有提供名字，Envoy 会生成一个 UUID 来作为它的名字。
