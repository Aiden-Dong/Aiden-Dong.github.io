---
layout:     post
title:      SSTable 文件格式 | leveldb 篇
subtitle:   
date:       2022-03-11
author:     Aiden
header-img: img/post-bg-mma-1.jpg
catalog: true  
tags:
    - olap
---

#### 前言

leveldb 中的每个sst主要有一下功能: 

1. 数据体 : 包含多个 key->value 的数据集合, sst 中的数据是按照 key, 由 **小到大进行排序**
2. 数据校验: 使用 `crc` 校验数据完整性
3. BloomFilter : 用来快速判断对于要查询的 key 是否在这个 sst 中
4. 数据索引: 对于要查询的key, 如果存在本 sst 中，则使用数据索引快速定位

#### 组成部分

`SST` 主要由 `5` 部分组成 : 

- **data_block** : 内部存储多个 key, value 的数据实体
- **filter_block** :  存储 `BloomFilter` 内容部分
- **meta_index_block** :  存储 `filter_block` 索引
- **index_block** : 存储 `data_block` 的索引
- **footer** : 存储 `index_block`, `meta_index_block` 的索引

```cpp
/*****
 * table_format :
 *              <beginning_of_file>
 *              [data block 1]
 *              [data block 2]
 *              ...
 *              [data block n]
 *              [filter block]
 *              [metaindex block]
 *              [index block]
 *              [Footer]        (fixed size; starts at file_size - sizeof(Footer))
 *              <end_of_file>
 */
```

![image.png]({{ site.url }}/assets/leveldb_1_1.png)

#### DataBlock

DataBlock 用来存储数据实体， 一个 SST 中可能会存在一到多个 `DataBlock`。

每个 `DataBlock` 中的数据按照 `key` 有序排序。

`DataBlock` 的数据格式如下 ：

```
 *      | shared_length(Varint32) | unshared_length(Varint32) | value_length(Varint32) | delta_key(string) | value(string) |
 *      | shared_length(Varint32) | unshared_length(Varint32) | value_length(Varint32) | delta_key(string) | value(string) |
 *      。。。。
 *      | shared_length(Varint32) | unshared_length(Varint32) | value_length(Varint32) | delta_key(string) | value(string) |
 *      | restarts_[0](Fixed32) | restarts_[1](Fixed32) | restarts_[2](Fixed32) | ... | restarts_[k](Fixed32) |
 *      | restarts_size(Fixed32) |
```

为了减少数据存储，每个Key-Value在 SST 中并不是独立存储，而是引入了共享key的概念。

每一个 Key-Value 存储格式为 : 

![image.png]({{ site.url }}/assets/leveldb_1_3.jpg)

- `shared_length` : 共享key长度
- `unshared_length` : 非共享 key 长度
- `value_length` : value 长度
- `delta_key` : 非共享 key内容
- `value` : value 内容

每一个 Key-Value 的获取需要首先知道上一个 key 的值， 然后基于 `shared_length`, `unshared_length`, `delta_key` 三个字段便可以求出这个 `key`.

DataBlock 的第一个 Key-Value 存储为 : 

```
shared_length    := 0
unshared_length  := len(key)
value_length     := len(value)
delta_key        := key
value            := value
```

同一个datablock中，每隔一部分需要重置共享Key(`shared_length    := 0`), 一个 datablock 中具有 `restart_number` 个重置点， 每个重置点的位置位于`restart_[]`数组中

```cpp
/***
 * buffer    : 保存要写入到磁盘的序列化字节流
 * restarts_ : 保存位于 buffer 的每个重置点的偏移位置
 */
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);     // 得到上次的key

  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty()  // No value
   || options_->comparator->Compare(key, last_key_piece) > 0);

  size_t shared = 0;

  // block_restart_interval 控制着重启点之间的距离
  if (counter_ < options_->block_restart_interval) {
    // 记录相同 key 的位置
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // Restart compression
    // 重启过程中 key 相当于置空 -> share=0
    restarts_.push_back(buffer_.size());  // 记录重启点的 buffer 偏移
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;  // 非共享 key 的长度

  // 存储数据长度
  PutVarint32(&buffer_, shared);       // 写入共享 key 长度   -- 变长 32 位
  PutVarint32(&buffer_, non_shared);   // 写入非共享 key 长度 -- 变长 32 位
  PutVarint32(&buffer_, value.size()); // 写入 value 长度    -- 变长 32 位

  // 存储真实数据
  buffer_.append(key.data() + shared, non_shared);   // 非共享 key 部分的存储
  buffer_.append(value.data(), value.size());           // 数据存储

  // 更新 last_key 表示为  上次插入的key
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);

  assert(Slice(last_key_) == key);
  counter_++;  // 计数加一
}
```

当数据写入完成后，会在最后追加重置点信息 

```cpp
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);             // 填充所有的重启点到 buffer
  }
  PutFixed32(&buffer_, restarts_.size());           // 填充重启点的数量到 buffer
  finished_ = true;
  return Slice(buffer_);                            // 返回整个 buffer
}
```

当一个 **DataBlock** 写入完成时， **IndexBlock**  会写入一下 **DataBlock** 的索引位置，写入 **IndexBlock** 的数据内容为 : 

**key**   : 写入到 **DataBlock** 最后一个 `key`
**value** : `DataBlock` 在 sst的位置(起始位置，长度)

#### FilterBlock

![image.png]({{ site.url }}/assets/leveldb_1_4.jpg)

**FilterBlock** 用来存储 **DataBlock**中的 `BloomFilter` 值。

它只有一个Block.

每当 **DataBlock** 刷盘时， 会构建一个 **BloomFilter**, 将这个 **DataBlock** 中所有的 key 加入到这个 `BloomFilter` 中。

所以有多少个 **DataBlock** 就有多少个 `BloomFilter`, 这些 `BloomFilter` 会拼接到一起。

他们使用 `filter_offsets_[]` 来记录下每个 bloomFilter 偏移。

```cpp
/***
 * Bloom 过滤器构建过程 ：
 * keys_           :  将所有的key都平铺拼接在一起
 * start_          :  每个 key 的偏移位置
 * result          :  存放得到的过滤器结果， 具有多个过滤器
 * filter_offsets_ : 每个过滤器的偏移量
 */
void FilterBlockBuilder::GenerateFilter() {

  const size_t num_keys = start_.size(); // 获取 key 的数量

  if (num_keys == 0) {
    filter_offsets_.push_back(result_.size());
    return;
  }

  // Make list of keys from flattened key structure
  start_.push_back(keys_.size());  // 因为每次都是填充上一次的key, 所以这次记录总得key的长度

  // 提取出所有的 key 使用  vector 存储
  tmp_keys_.resize(num_keys);
  for (size_t i = 0; i < num_keys; i++) {
    const char* base = keys_.data() + start_[i];
    size_t length = start_[i + 1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

  // 存储上次 Bloom 过滤器写入后的数据大小
  filter_offsets_.push_back(result_.size());

  // 计算 Bloom 过滤器的值， 并追加到 result 中
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  // 清空 key
  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}
```

当所有的 **DataBlock** 落盘后， **FilterBlock** 构造完成开始落盘 : 

```cpp
Slice FilterBlockBuilder::Finish() {

  // 如果有 key 没有还没有计算完bloomFilter 则触发计算
  if (!start_.empty()) {
    GenerateFilter();
  }

  // Append array of per-filter offsets
  const uint32_t array_offset = result_.size();
  for (size_t i = 0; i < filter_offsets_.size(); i++) {
    // 填充每个bloom过滤器的偏移
    PutFixed32(&result_, filter_offsets_[i]);
  }

  // 填充总的Bloom过滤器的长度
  PutFixed32(&result_, array_offset);
  // 填充 11
  result_.push_back(kFilterBaseLg);  // Save encoding parameter in result
  return Slice(result_);
}
```

为了提高基于 **DataBlock**定位 `BloomFilter` 的速度， 并不是 `BloomFilter`的数量与 `filter_offsets_` 的个数一一对应。
每次 **DataBlock** 写完, 假设下次写入的 **DataBlock**偏移量为 `next_datablock_offset`。

则这次 **DataBlock** 的 `filter_offsets_` 要填充`(next_datablock_offset/1<<11)-1`次，保证下个**DataBlock**的 `BloomFilter` 从 `(next_datablock_offset/1<<11)`处开始填充。

这样我们拿到 **DataBlock** 可以基于其偏移位置 `datablock_offset`, 通过 `datablock_offset/1<<11` 直接定位到对应 `BloomFilter` 的 `filter_offsets_` 位置。

```cpp
void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
  uint64_t filter_index = (block_offset / kFilterBase);  // 现在需要的 filter_offset 数量
  assert(filter_index >= filter_offsets_.size());

  // 存在的意义是为了多写几个无效的偏移
  // 然后用于快速基于 block_offset 快速定位 filter_offsets
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
}
```

读取逻辑 :

```cpp
bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {

  // 基于 block_offset 定位到 offset_[index]
  uint64_t index = block_offset >> base_lg_;  // 等同于 block_offset/kFilterBase

  if (index < num_) {
    // 截取 Bloom 过滤器
    uint32_t start = DecodeFixed32(offset_ + index * 4);
    uint32_t limit = DecodeFixed32(offset_ + index * 4 + 4);

    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
      Slice filter = Slice(data_ + start, limit - start);
      // 匹配查找
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // Empty filters do not match any keys
      return false;
    }
  }
  return true;  // Errors are treated as potential matches
}
```

#### MetaIndexBlock

**MetaIndexBlock** 主要用来记录 **FilterBlock** 的位置， 充当 **FilterBlock** 的索引

![image.png]({{ site.url }}/assets/leveldb_1_5.jpg)

它的数据结构跟 DataBlock 一样，不过他只有一个 Key-Value， 只有一个Block.

- **Key** : `filter.leveldb.BuiltinBloomFilter2`
- **Value** : BloomFilter 在文件中的索引(起始位置，大小)


#### IndexBlock 

**IndexBlock** 数据结构与 **DataBlock** 相同，它有多个 Key-Value 组成，只有一个Block.


- **Key** : 每个 **DataBlock** 最后一个key(类似这个逻辑，但不是完全这样)
- **Value** : **DataBlock** 的索引(起始位置，大小)

#### Footer

Footer 在SST 的末尾， 他由固定的 `48 字节`构成。

![image.png]({{ site.url }}/assets/leveldb_1_6.jpg)

第一部分为 **MetaIndexBlock** 的索引构成(起始位置，大小)，
第二部分为 **IndexBlock**的索引构成(起始位置，大小)。

如果这两部分加起来长度不足40，则补零填充到`40字节`。

最后填充8字节的幻数，完成SST的填充。

#### SST 的写入过程

写入过程在 `table_builder.cc` 里面有说明 : 


首先为写 **DataBlock** 过程

```cpp
***
 * 数据写入 sstable：
 *  1. key 的写入顺序为有序， 依次递增。
 *  2. 写入 datablock 之前首先写入 filter_block
 *  3. 写入datablock
 *  4. 如果 datablock 超过限制，进行datablock 刷盘操作
 *  5. 如果刷盘操作，在 index_block 中记录上个 datablock 的索引位置
 *
 * @param key
 * @param value
 */
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;      // 获得引用
  assert(!r->closed);
  if (!ok()) return;   // 记录写入状态

  if (r->num_entries > 0) {
    // 要求排序递增
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  if (r->pending_index_entry) {
    /**
     * 上一个 datablock 写入完成， 需要记录一下 datablock 的index 信息
     * index_block 数据存储：
     *      格式 : block_builder
     *           key   : 上次写入的key(r->last_key)
     *           value : pending_handle(上一个datablock 写入前的offset, 上一个datablock size)
     *
     */
    assert(r->data_block.empty());
    // 比较并截断 last_key
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);              // 序列化出偏移
    r->index_block.Add(r->last_key, Slice(handle_encoding));   // 记录last key 的偏移位置
    r->pending_index_entry = false;
  }

  if (r->filter_block != nullptr) {
    // Bloom 过滤器需要写入 key
    r->filter_block->AddKey(key);
  }

  // 将插入的 key 标志入 last_key
  r->last_key.assign(key.data(), key.size());
  r->num_entries++;                 // 数据元素增一
  r->data_block.Add(key, value);    // datablock 插入元素

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();  // 计算 datablock 当前数据量的大小

  // 如果 datablock 数据量的大小超过设置的 block_size
  // 则 进行刷写操作
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

可以看出如果 **DataBlock** 超过一定的大小，则刷盘，重新写一个 **DataBlock**,
这时在 **IndexBlock** 中记录一下写完成 **DataBlock** 的索引位置。

等到所有的 Key-Value 都写完以后:

```cpp
/***
 * 数据写入完成， 进行归档操作 ：
 *    | data_block_1 | data_block_2 | ... | data_block_n |
 *    | filter_block |
 *    | meta_index_block |
 *    | index_block |
 *    | footer |
 * @return
 */
Status TableBuilder::Finish() {
  Rep* r = rep_;

  // Data Block 剩余数据落盘
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Filter Block 数据落盘
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression, &filter_block_handle);
  }

  // Meta index block 数据落盘
  // key   : 过滤器名称
  // value : 过滤器 BlockHandle
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);

    if (r->filter_block != nullptr) {
      // 记录过滤器名称
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());

      // 记录过滤器的位置索引信息
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // IndexBlock 数据落盘
  // key   : last_key
  // value : last_key 对应的 datablock 的 BlockHandle
  if (ok()) {

    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }

    WriteBlock(&r->index_block, &index_block_handle);
  }

  // 写 Footer 数据落盘
  // metaindex_block_handle 元信息记录
  // index_block_handle     元信息记录
  // kTableMagicNumber      数据落地
  if (ok()) {
    // 填充 Footer 数据
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);

    // 序列化数据
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);

    // 写入文件
    r->status = r->file->Append(footer_encoding);

    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }

  return r->status;
}
```

由此完成了一个 SST 的数据写入。

#### 数据读取


##### 每个部分的 Block 的数据读取过程

**IndexBlock**, **MetaIndexBlock**,**DataBlock** 的数据解析都是通过 `Block::Iter`, 它通过 **二分查找** ，快速定位key, 并获取Value.

它首先基于每个重置索引，在所有的重置点基于二分查找，比较每个重置点第一个key(因为第一个key是完整的)。粗略定位到 key 的大致位置， 然后在缩小范围，精确查找 Key，Value.

```cpp
/***
   * 使用二分查找法， 遍历 block 试图找到对应的 key
   */
  void Seek(const Slice& target) override {
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;                      // 偏移点下限
    uint32_t right = num_restarts_ - 1;     // 偏移点上限

    int current_key_compare = 0;

    if (Valid()) {
      // 表示存在有效数据， 利用上次查询加速这次数据查询过程
      current_key_compare = Compare(key_, target);
      if (current_key_compare < 0) {
        left = restart_index_;
      } else if (current_key_compare > 0) {
        right = restart_index_;
      } else {
        return;
      }
    }

     // 使用二分查找
     //
     // 每次只跟每个重启块 的 第一个Key进行比较
     // 因为第一个key是完整的， 所以比较速度快
    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;

      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;

      // 获取 data[mid] 的 block 的首 key
      const char* key_ptr = DecodeEntry(data_ + region_offset, data_ + restarts_, &shared, &non_shared, &value_length);

      if (key_ptr == nullptr || (shared != 0)) {
        CorruptionError();
        return;
      }

      Slice mid_key(key_ptr, non_shared);

      if (Compare(mid_key, target) < 0) {
        // 数据在右边
        left = mid;
      } else {
        // 数据在左边
        right = mid - 1;
      }
    }

    // left == right == mid || Compare(mid_key) == 0

    // We might be able to use our current position within the restart block.
    // This is true if we determined the key we desire is in the current block and is after than the current key.
    assert(current_key_compare == 0 || Valid());
    bool skip_seek = left == restart_index_ && current_key_compare < 0;  // false

    if (!skip_seek) {
      SeekToRestartPoint(left);
    }
    // Linear search (within restart block) for first key >= target
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      // 可以 从小到达排序
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }
```

整个数据块(**IndexBlock**, **MetaIndexBlock**或某个**DataBlock**) 首先通过已知的偏移位置通过 `RandomAccessFile` 从文件读取到内存中，形成一个 `BlockContents`, 它主要是将序列化的数据流读取到内存中形成的一个字节数组。

而后基于数据块的数据结构， 我们封装了这个 `Block` 类， 用来解析并查询这个数据块。

```cpp
/***
 * 从文件中读取一个 Block 并 填充到 BlockContents 中
 * 
 * @param file    sst 文件
 * @param options 文件读取选项
 * @param handle  Block 的偏移元信息
 * @param result  将Block 转换为一个  BlockContests
 */
Status ReadBlock(RandomAccessFile* file, const ReadOptions& options, const BlockHandle& handle, BlockContents* result) {

  result->data = Slice();
  result->cachable = false;
  result->heap_allocated = false;

  // kBlockTrailerSize 是 压缩标识 + crc校验 
  size_t n = static_cast<size_t>(handle.size());
  char* buf = new char[n + kBlockTrailerSize];
  Slice contents;
  Status s = file->Read(handle.offset(), n + kBlockTrailerSize, &contents, buf);
  if (!s.ok()) {
    delete[] buf;
    return s;
  }

  if (contents.size() != n + kBlockTrailerSize) {
    delete[] buf;
    return Status::Corruption("truncated block read");
  }


  const char* data = contents.data();  

  // 数据体的校验
  if (options.verify_checksums) {
    const uint32_t crc = crc32c::Unmask(DecodeFixed32(data + n + 1));
    const uint32_t actual = crc32c::Value(data, n + 1);
    if (actual != crc) {
      delete[] buf;
      s = Status::Corruption("block checksum mismatch");
      return s;
    }
  }

  // 获取压缩类型，如果存在压缩， 变执行压缩操作
  switch (data[n]) {
    case kNoCompression:
      if (data != buf) {
        delete[] buf;
        result->data = Slice(data, n);
        result->heap_allocated = false;
        result->cachable = false;  // Do not double-cache
      } else {
        result->data = Slice(buf, n);
        result->heap_allocated = true;
        result->cachable = true;
      }
      break;
    case kSnappyCompression: {
      size_t ulength = 0;
      if (!port::Snappy_GetUncompressedLength(data, n, &ulength)) {
        delete[] buf;
        return Status::Corruption("corrupted compressed block contents");
      }
      char* ubuf = new char[ulength];
      if (!port::Snappy_Uncompress(data, n, ubuf)) {
        delete[] buf;
        delete[] ubuf;
        return Status::Corruption("corrupted compressed block contents");
      }
      delete[] buf;
      result->data = Slice(ubuf, ulength);
      result->heap_allocated = true;
      result->cachable = true;
      break;
    }
    default:
      delete[] buf;
      return Status::Corruption("bad block type");
  }

  return Status::OK();
}


/***
 * BlockContext 到 Block 的转换
 */
 Block::Block(const BlockContents& contents)
    : data_(contents.data.data()),
      size_(contents.data.size()),
      owned_(contents.heap_allocated) {

  if (size_ < sizeof(uint32_t)) {
    size_ = 0;  // Error marker
  } else {
    // 模糊计算最多有多少个重启点
    size_t max_restarts_allowed = (size_ - sizeof(uint32_t)) / sizeof(uint32_t);

    if (NumRestarts() > max_restarts_allowed) {
      // The size is too small for NumRestarts()
      size_ = 0;
    } else {
      restart_offset_ = size_ - (1 + NumRestarts()) * sizeof(uint32_t);
    }
  }
}
```

##### Table 

SST 的读取主要通过 `Table` 类来完成， 它内部提供了 `InternalGet` 方法在这个SST上进行查询用户请求的`key`:

`Status InternalGet(const ReadOptions&, const Slice& key, void* arg, void (*handle_result)(void* arg, const Slice& k, const Slice& v))` 

> 它的查询过程 

1. 查询 IndexBlock 找到所有的 DataBlock的位置信息
2. 对于每个要查询的DataBlock，首先使用 `BloomFilter` (位于FilterBlock)判断这个数据是否在这个DataBlock中。
3. 如果判断数据存在， 然后加载这个DataBlock, 使用`Block` 来二分查找这个key.

> 说明 :

**DataBlock 的加载过程维护了一个 LRU 的缓存机制， 对于要加载的DataBlock, Table 首先要去缓存中查看这个Block是否存在，如果不存在，则从文件中读取，并且加载到缓存中，以便于后续的读取。**

```cpp
/***
 * 基于回调的方式， 查找某个 key, 如果找到则调用 handle_result(arg, key, value)
 *
 * @param options            配置选项
 * @param k                  key
 * @param arg                参数
 * @param handle_result      找到外部数据时候触发的外部调用
 *
 */
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg, void (*handle_result)(void*, const Slice&,const Slice&)) {
  Status s;

  // IndexBlock 迭代器
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  iiter->Seek(k);   // key_ >= k

  if (iiter->Valid()) {  // 找到这个数据
    Slice handle_value = iiter->value();         // 定位到 datablock 的位置
    FilterBlockReader* filter = rep_->filter;    // 获得过滤器
    BlockHandle handle;

    // 使用 Bloom 过滤器判断一下数据是否在这个 block 里面
    if (filter != nullptr && handle.DecodeFrom(&handle_value).ok() && !filter->KeyMayMatch(handle.offset(), k)) {
      // Bloom 过滤器没有找到这个数据
    } else {
      // 加载这个 block
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {
        // 找到这个数据，调用回调函数
        (*handle_result)(arg, block_iter->key(), block_iter->value());
      }
      s = block_iter->status();
      delete block_iter;
    }
  }

  if (s.ok()) {
    s = iiter->status();
  }
  delete iiter;
  return s;
}
```

对于 SST 的数据查询与遍历， `Table` 提供了 `TwoLevelIterator`遍历器查询器。

`TwoLevelIterator` 是一个基于两层索引的数据查询器 : 

- 第一层为IndexBlock 的 DataBlock 索引
- 第二层为DataBlock 的具体数据索引

数据查询与遍历首先通过 IndexBlock 进行粗略定位， 如果定位成功以后，则使用 DataBlock 进行进一步查询。

示例 : 

```cpp
void TwoLevelIterator::Seek(const Slice& target) {
  // key >= target, 因为从最小的 key 遍历上来的
  // 所以如果存在， 肯定在这里
  index_iter_.Seek(target);

  InitDataBlock();  // 定位到指定的 datablock

  if (data_iter_.iter() != nullptr) data_iter_.Seek(target);  // 从当前 datablock 中定位到 target

  SkipEmptyDataBlocksForward();
}
```

#### UML

![image.png]({{ site.url }}/assets/leveldb_1_2.png)