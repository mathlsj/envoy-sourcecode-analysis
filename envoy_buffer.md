# Envoy 源码分析－－buffer

>申明：本文的 Envoy 源码分析基于 Envoy1.10.0。

`Envoy` 的 `buffer` 在 1.10.0 前是基于 libevent 的 `evbuffer` 进行封装。在 1.10.0 开始为了提高性能，要使用 libev 或 libuv 来替代 libevent 重写了个 `buffer` 来消除 `evbuffer` 的依赖。想要具体了解可看 [issue#4952](https://github.com/envoyproxy/envoy/issues/4952) 和 [issue#5441](https://github.com/envoyproxy/envoy/pull/5441)。下面我们先来看下 `buffer` 相关的类图：

![envoy_buffer](./images/envoy_buffer.png)

上面四个 `slice` 相关的类是重写后的 `buffer`。`slice` 封装了 buffer 相关操作，`OwnedSlice` 是创建 `slice` 的类，`SliceDeque` 管理 `OwnedSlice` 操作队列，`UnownedSlice` 则是兼容 `BufferFragment` 的类。

`LibEventInstance` 继承自 `Instance` 对 `Instance` 的接口进行扩充新增了两个接口。`OwnedImpl` 对 `LibEventInstance` 接口的实现。`WatermarkBuffer` 则在 `OwnedImpl` 的基础上增加水位的高低告警回调。`WatermarkBufferFactory` 继承 `WatermarkFactory` 只是生成 `WatermarkBuffer` 的适配器。


## BufferFragment

针对外部数据和大小创建一个适配器，这个类不拷贝内存，只是一个指针，外部需要自己确定数据是有效的。调用 `done()` 会释放数据。

代码详情如下：

```
class BufferFragmentImpl : NonCopyable, public BufferFragment {
public:
  BufferFragmentImpl(
      const void* data, size_t size,
      const std::function<void(const void*, size_t, const BufferFragmentImpl*)>& releasor)
      : data_(data), size_(size), releasor_(releasor) {}

  // Buffer::BufferFragment
  const void* data() const override { return data_; }
  size_t size() const override { return size_; }
  void done() override {
    if (releasor_) {
      releasor_(data_, size_, this);
    }
  }

private:
  const void* const data_;
  const size_t size_;
  const std::function<void(const void*, size_t, const BufferFragmentImpl*)> releasor_;
};
```

## RawSlice

只是一个内存切片的数据结构体。
```
struct RawSlice {
  void* mem_ = nullptr;
  size_t len_ = 0;

  bool operator==(const RawSlice& rhs) const { return mem_ == rhs.mem_ && len_ == rhs.len_; }
};
```

## Slice

`Slice` 是一个连续的内存块的管理。其块的管理方式如下：

```
 *                   |<- data_size() -->|<- reservable_size() ->|
 * +-----------------+------------------+-----------------------+
 * | Drained         | Data             | Reservable            |
 * | Unused space    | Usable content   | New content can be    |
 * | that formerly   |                  | added here with       |
 * | was in the Data |                  | reserve()/commit()    |
 * | section         |                  |                       |
 * +-----------------+------------------+-----------------------+
 *                   ^
 *                   |
 *                   data()
```

前面是个无用内存块，中间这块就是数据自身，后面是还可用的内存块长度。增加数据支持 `prepend` 和 `append` 即可追加数据也可以前置增加。

`Slice` 的数据存储方式如下：

```
  /** 切片指针 */
  uint8_t* base_{nullptr};

  /** 存储数据相对于切片的偏移量 */
  uint64_t data_;

  /** 可用空间相对于切片的偏移量 */
  uint64_t reservable_;

  /** 切片总长度 */
  uint64_t capacity_;

  /** 是否被`Limter`限制以延迟操作 */
  bool reservation_outstanding_{false};
```

现在来看几个主要的操作。

`append` 在后面追加数据。增加尽量多的数量，不保证所有数据都增加成功。这个在空间不够时，没有重新申请空间。先判断数据是否被锁住了，接着取可用长度和增加长度的最小值，可用空间偏移量后移，copy 数据到块中。

```
  uint64_t append(const void* data, uint64_t size) {
    if (reservation_outstanding_) {
      return 0;
    }
    uint64_t copy_size = std::min(size, reservableSize());
    uint8_t* dest = base_ + reservable_;
    reservable_ += copy_size;
    memcpy(dest, data, copy_size);
    return copy_size;
  }
```

`prepend` 前置增加数据。同 `append` 一样没有申请空间，只是增加尽量多的数据，不保证所有数据都增加成功。先判断数据是否被锁住了，然后判断切片内是否有数据。如果没有数据直接增加到最后面，加在最后面的目的是为了后面继续 `prepend` 时有空间可加（这里没想明白，为啥要在最后面加，这样我下次用 `append` 不也一样没法增加数据）。后面取出前置可用长度和增加长度的最小值，`data` 的偏移量后移， copy 数据块。

```
  uint64_t prepend(const void* data, uint64_t size) {
    if (reservation_outstanding_) {
      return 0;
    }
    const uint8_t* src = static_cast<const uint8_t*>(data);
    uint64_t copy_size;
    if (dataSize() == 0) {
      copy_size = std::min(size, reservableSize());
      reservable_ = capacity_;
      data_ = capacity_ - copy_size;
    } else {
      if (data_ == 0) {
        return 0;
      }
      copy_size = std::min(size, data_);
      data_ -= copy_size;
    }
    memcpy(base_ + data_, src + size - copy_size, copy_size);
    return copy_size;
  }
```

`drain` 删除数据，这个只是偏移量后移。

```
  void drain(uint64_t size) {
    ASSERT(data_ + size <= reservable_);
    data_ += size;
    if (data_ == reservable_ && !reservation_outstanding_) {
      data_ = reservable_ = 0;
    }
  }
```

`reserve` 和 `commit` 是合起来一起用的。`reserve` 返回可用内存块同时将切片内存块锁住，不让增加和移除，直到 `commit` 才恢复正常。`reserve` 返回的内存块可由用户自己填充。`reserve` 时如果增加进来的内存块不是同个切片下的会增加失败。

```
  Reservation reserve(uint64_t size) {
    if (reservation_outstanding_ || size == 0) {
      return {nullptr, 0};
    }
    uint64_t available_size = capacity_ - reservable_;
    if (available_size == 0) {
      return {nullptr, 0};
    }
    uint64_t reservation_size = std::min(size, available_size);
    void* reservation = &(base_[reservable_]);
    reservation_outstanding_ = true;
    return {reservation, reservation_size};
  }

  bool commit(const Reservation& reservation) {
    if (static_cast<const uint8_t*>(reservation.mem_) != base_ + reservable_ ||
        reservable_ + reservation.len_ > capacity_ || reservable_ >= capacity_) {
      // The reservation is not from this OwnedSlice.
      return false;
    }
    ASSERT(reservation_outstanding_);
    reservable_ += reservation.len_;
    reservation_outstanding_ = false;
    return true;
  }
```

## OwnedSlice

只是创建和删除 `Slice` 指针。

```
  static SlicePtr create(uint64_t capacity) {
    uint64_t slice_capacity = sliceSize(capacity);
    return SlicePtr(new (slice_capacity) OwnedSlice(slice_capacity));
  }

  static void operator delete(void* address) { ::operator delete(address); }
```

## SliceDeque

`Slice` 管理队列。支持前后置增加和删除。默认队列长度 8，当长度不够用时，会成倍自增长。

这代码比较简单。这里只列出自增长函数。

```
  void growRing() {
    if (size_ < capacity_) {
      return;
    }
    const size_t new_capacity = capacity_ * 2;
    auto new_ring = std::make_unique<SlicePtr[]>(new_capacity);
    for (size_t i = 0; i < new_capacity; i++) {
      ASSERT(new_ring[i] == nullptr);
    }
    size_t src = start_;
    size_t dst = 0;
    for (size_t i = 0; i < size_; i++) {
      new_ring[dst++] = std::move(ring_[src++]);
      if (src == capacity_) {
        src = 0;
      }
    }
    for (size_t i = 0; i < capacity_; i++) {
      ASSERT(ring_[i].get() == nullptr);
    }
    external_ring_.swap(new_ring);
    ring_ = external_ring_.get();
    start_ = 0;
    capacity_ = new_capacity;
  }
```

## UnownedSlice

管理 `BufferFragment` 的特殊 `Slice`。

```
class UnownedSlice : public Slice {
public:
  UnownedSlice(BufferFragment& fragment)
      : Slice(0, fragment.size(), fragment.size()), fragment_(fragment) {
    base_ = static_cast<uint8_t*>(const_cast<void*>(fragment.data()));
  }
  ~UnownedSlice() override { fragment_.done(); }
private:
  BufferFragment& fragment_;
};
```

## OwnedImpl

`OwnedImpl` 是 `LibEventInstance` 接口的实现，同时为了支持新 `buffer` 在数据操作时，会有一个 bool 值来判断调用的是新 `buffer` 还是旧 `buffer`。新的 `buffer` 在 1.10.0 默认情况下是关闭的，可通过配置 `–use-libevent-buffers 0` 打开。

```add
OwnedImpl::OwnedImpl() : old_impl_(use_old_impl_) {
  if (old_impl_) {
    buffer_ = evbuffer_new();
  }
}
```

现在我们在分析下几个重要的操作：

`add` 增加数据。几个 `add` 的操作最终都是调用 add(const void* data, uint64_t size)。

```
void OwnedImpl::add(const void* data, uint64_t size) {
  //判断是否为旧的buffer
  if (old_impl_) {
    //直接调用
    evbuffer_add(buffer_.get(), data, size);
  } else {
    const char* src = static_cast<const char*>(data);
    //检查是否需要新加切片。如果切片为空，需要新加。
    bool new_slice_needed = slices_.empty();
    //循环加入。数据加入切片，一个切片并不一定能把数据全都加进去，所以需要循环加入。
    while (size != 0) {
      if (new_slice_needed) {
        //创建切片并加入切片队列
        slices_.emplace_back(OwnedSlice::create(size));
      }
      //切片加入数据并返回加入的长度
      uint64_t copy_size = slices_.back()->append(src, size);
      //增加偏移量
      src += copy_size;
      //看是否还有数据未加入。size 大于 0 说明上个切片已满，需要重新创建切片。
      size -= copy_size;
      length_ += copy_size;
      new_slice_needed = true;
    }
  }
}
```

`prepend` 这个和 `add` 接口类似，需要的自行看源码。

`read` 从 `IoHandle` 中读取数据，并加入 `buffer`。

```
Api::IoCallUint64Result OwnedImpl::read(Network::IoHandle& io_handle, uint64_t max_length) {
  //长度判断，一次读取最大长度不能为0.
  if (max_length == 0) {
    return Api::ioCallUint64ResultNoError();
  }
  //申明一个RawSlice
  constexpr uint64_t MaxSlices = 2;
  RawSlice slices[MaxSlices];
  //锁住操作的内存块
  const uint64_t num_slices = reserve(max_length, slices, MaxSlices);
  //readv读取数据，数据存在slices
  Api::IoCallUint64Result result = io_handle.readv(max_length, slices, num_slices);
  //旧buffer
  if (old_impl_) {
    if (!result.ok()) {
      //错误直接返回
      return result;
    }
    uint64_t num_slices_to_commit = 0;
    //读取到的长度
    uint64_t bytes_to_commit = result.rc_;
    ASSERT(bytes_to_commit <= max_length);
    //RawSlice检测，检查slices有几个。
    while (bytes_to_commit != 0) {
      //空间判断，看下一次能加多少数据长度。
      slices[num_slices_to_commit].len_ =
          std::min(slices[num_slices_to_commit].len_, static_cast<size_t>(bytes_to_commit));  
      ASSERT(bytes_to_commit >= slices[num_slices_to_commit].len_);
      bytes_to_commit -= slices[num_slices_to_commit].len_;
      num_slices_to_commit++;
    }
    //不能大于RawSlice本身数组长度，否则直接断言。
    ASSERT(num_slices_to_commit <= num_slices);
    //提交数据到buffer
    commit(slices, num_slices_to_commit);
  //新buffer
  } else {
    uint64_t bytes_to_commit = result.ok() ? result.rc_ : 0;
    ASSERT(bytes_to_commit <= max_length);
    //新的buffer reserve操作返回的是slice的个数。循环判断。决定 Slice的长度。
    for (uint64_t i = 0; i < num_slices; i++) {
      slices[i].len_ = std::min(slices[i].len_, static_cast<size_t>(bytes_to_commit));
      bytes_to_commit -= slices[i].len_;
    }
    //提交数据到buffer
    commit(slices, num_slices);
  }
  return result;
}
```

`write` 从 `buffer` 中读数据。调用writev。

```
Api::IoCallUint64Result OwnedImpl::write(Network::IoHandle& io_handle) {
  constexpr uint64_t MaxSlices = 16;
  RawSlice slices[MaxSlices];
  //获取写入的数据和大小。
  const uint64_t num_slices = std::min(getRawSlices(slices, MaxSlices), MaxSlices);
  //写入数据
  Api::IoCallUint64Result result = io_handle.writev(slices, num_slices);
  if (result.ok() && result.rc_ > 0) {
    //写入成功，将数据从buffer中清除。
    drain(static_cast<uint64_t>(result.rc_));
  }search
  return result;
}
```

## WatermarkBuffer

`WatermarkBuffer` 是在 `OwnedImpl` 的基础上增加了高低水位告警回调，在代码还是调用了 `OwnedImpl`。

```
  std::function<void()> below_low_watermark_;
  std::function<void()> above_high_watermark_;

  // Used for enforcing buffer limits (off by default). If these are set to non-zero by a call to
  // setWatermarks() the watermark callbacks will be called as described above.
  uint32_t high_watermark_{0};
  uint32_t low_watermark_{0};
```

我们来看其中一个函数：

```
Api::IoCallUint64Result WatermarkBuffer::read(Network::IoHandle& io_handle, uint64_t max_length) {
  //调用OwnedImpl的函数
  Api::IoCallUint64Result result = OwnedImpl::read(io_handle, max_length);
  //高水位检测
  checkHighWatermark();
  return result;
}
```

## WatermarkBufferFactory

`WatermarkBuffer` 的适配器，只是用来创建 `WatermarkBuffer` 的。

## ZeroCopyInputStreamImpl

`ZeroCopyInputStreamImpl` 可以把它认为是 `Buffer::InstancePtr` 的一个迭代器。

```
  //数据在buffer中的位移
  uint64_t position_{0};
  //结束标志
  bool finished_{false};
  //NEXT过的所有字节数。
  uint64_t byte_count_{0};
```