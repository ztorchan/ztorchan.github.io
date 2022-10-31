---
title: RocksDB的Put流程
tags:
  - rocksdb
  - 源码剖析
categories:
  - RocksDB源码阅读
date: 2022-10-21 22:13:24
---


今天这篇文章从源码自顶向下地梳理一下RocksDB的写入流程。需要强调的是，RocksDB的写入流程非常复杂，本文的主要目的是记录从用户调用`Put`到完成写入过程中数据流动和调用关系，因此忽略了许多机制，仅保留了主干过程。

<!-- more -->

## 用户接口
首先当然是从用户调用的接口开始探索。文档里给出了RocksDB的写入调用形式：
``` C++ 用户调用示例
  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());

  std::string key, value;
  rocksdb::Status s = db->Put(rocksdb::WriteOptions(), key, value);
  s = db->Delete(rocksdb::WriteOptions(), key);
```
可以看到用户通过`rocksdb::DB`的`Put`方法写入，`Delete`方法删除。我们暂且忽略`Delete`，追查下去看看`Put`究竟做了什么。

## `DB`和`DBImpl`的Put方法
`rocksdb::DB`类（由于RocksDB统一使用了`rocksdb`的命名空间，接下来的描述将不再加上`rocksdb::`）的声明被放在了`/include/db.h`里，其中关于`Put`方法的声明如下：
``` C++
class DB {
  ...

  // Set the database entry for "key" to "value".
  // If "key" already exists, it will be overwritten.
  // Returns OK on success, and a non-OK status on error.
  // Note: consider setting options.sync = true.
  virtual Status Put(const WriteOptions& options,
                     ColumnFamilyHandle* column_family, const Slice& key,
                     const Slice& value) = 0;
  virtual Status Put(const WriteOptions& options,
                     ColumnFamilyHandle* column_family, const Slice& key,
                     const Slice& ts, const Slice& value) = 0;
  virtual Status Put(const WriteOptions& options, const Slice& key,
                     const Slice& value) {
    return Put(options, DefaultColumnFamily(), key, value);
  }
  virtual Status Put(const WriteOptions& options, const Slice& key,
                     const Slice& ts, const Slice& value) {
    return Put(options, DefaultColumnFamily(), key, ts, value);
  }
  
  // 需要说明的是，这里的Slice是RocksDB对封装的一种字符串，
  // 在本文中把它看作普通的字符串即可。
  ...
}
```
`Put`方法一共有四个重载。第一个需要提供的参数：写入设置引用`const WriteOptions&`、列族handle指针`ColumnFamilyHandle*`以及需要存储的键和值`const Slice&`。第二个在第一个的基础上添加了时间戳`const Slice&`。第三个和第四个不过是用户省略列族handle指针，由其提供默认列族，调用回第一个和第二个`Put`方法。值得注意的是，前两个`Put`方法是纯虚函数，也就是说还有一个子类是以`DB`为基类构建的，是真正供以实例化的类。  

那么这个类在哪呢？`/db/db_impl.h`里声明了`DB`的实现类`DBImpl`：
``` C++
class DBImpl : public DB {
  ...
  using DB::Put;
  Status Put(const WriteOptions& options, ColumnFamilyHandle* column_family,
             const Slice& key, const Slice& value) override;
  Status Put(const WriteOptions& options, ColumnFamilyHandle* column_family,
             const Slice& key, const Slice& ts, const Slice& value) override;
  ...
}
```
`DBImpl`暴露了父类`DB`的四个`Put`方法，并声明了两个相同的带列族参数的`Put`方法（即和`DB`前两个`Put`完全相同）。首先来看看`DBImpl::Put`的定义，RocksDB在`/db/db_impl_write.cc`对它们进行了实现：
``` C++
// DBImpl的Put方法
Status DBImpl::Put(const WriteOptions& o, ColumnFamilyHandle* column_family,
                   const Slice& key, const Slice& val) {
  const Status s = FailIfCfHasTs(column_family);
  if (!s.ok()) {
    return s;
  }
  return DB::Put(o, column_family, key, val);
}

Status DBImpl::Put(const WriteOptions& o, ColumnFamilyHandle* column_family,
                   const Slice& key, const Slice& ts, const Slice& val) {
  const Status s = FailIfTsMismatchCf(column_family, ts, /*ts_for_read=*/false);
  if (!s.ok()) {
    return s;
  }
  return DB::Put(o, column_family, key, ts, val);
}
```
很明显，`DBImpl::Put`也只是在对列族和时间戳作了合法性检查之后再调用回前两个`DB::Put`执行真正的写入工作。所以，为了继续探究写流程，只要沿着前两个`DB::Put`的实现继续深入就好了。对`DB::Put`的实现同样在`/db/db_impl_write.cc`中，定义如下：
``` C++
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24, 0 /* max_bytes */,
                   opt.protection_bytes_per_key, 0 /* default_cf_ts_sz */);
  Status s = batch.Put(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}

Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& ts, const Slice& value) {
  ColumnFamilyHandle* default_cf = DefaultColumnFamily();
  assert(default_cf);
  const Comparator* const default_cf_ucmp = default_cf->GetComparator();
  assert(default_cf_ucmp);
  WriteBatch batch(0 /* reserved_bytes */, 0 /* max_bytes */,
                   opt.protection_bytes_per_key,
                   default_cf_ucmp->timestamp_size());
  Status s = batch.Put(column_family, key, ts, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}
```
**在真正进行写入工作时，`Put`还是先用`WriteBatch`对其进行了封装，再调用`Write`写入**。和平常用户使用`WriteBatch`进行批量原子更新是一样的，只不过这个batch中只有一个Put操作。所以接下来，我们就要看看`Write`是怎样把`WriteBatch`写入的。

## `WriteBatch`和`Write`
简单起见，从这一章开始只对不带时间戳的写流程继续追查下去（更多是我还没完全理解RocksDB中时间戳的概念，或许是和Bigtable中的是一致的？）。也就是从这一个方法开始：
``` C++
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24, 0 /* max_bytes */,
                   opt.protection_bytes_per_key, 0 /* default_cf_ts_sz */);
  Status s = batch.Put(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}
```
这里的逻辑非常简单: 
1. 创建了一个`WriteBatch`
2. `WriteBatch`调用`Put`记录操作
3. 调用`Write`把`WriteBatch`更新写入数据库  


### `WriteBatch`的定义
鉴于本文目的在于尽可能简单地梳理一遍写流程，源码仅挑关键的部分展示和解释。关于`WriteBatch`的声明在`include/rocksdb/write_batch.h`：
``` C++
class WriteBatch : public WriteBatchBase {
  ......

  protected:
    std::string rep_;
}
```
`WriteBatch`继承于基类`WriteBatchBase`，基类是一个完全由虚函数构成的接口类。`rep_`是`WriteBatch`最重要的部分，它是一个字符串，按一定的格式记录了所要进行所有操作，记录格式如下：
```
WriteBatch::rep_ :=
   sequence: fixed64
   count: fixed32
   data: record[count]
record :=
   kTypeValue varstring varstring
   kTypeDeletion varstring
   kTypeSingleDeletion varstring
   kTypeRangeDeletion varstring varstring
   kTypeMerge varstring varstring
   kTypeColumnFamilyValue varint32 varstring varstring
   kTypeColumnFamilyDeletion varint32 varstring
   kTypeColumnFamilySingleDeletion varint32 varstring
   kTypeColumnFamilyRangeDeletion varint32 varstring varstring
   kTypeColumnFamilyMerge varint32 varstring varstring
   kTypeBeginPrepareXID
   kTypeEndPrepareXID varstring
   kTypeCommitXID varstring
   kTypeCommitXIDAndTimestamp varstring varstring
   kTypeRollbackXID varstring
   kTypeBeginPersistedPrepareXID
   kTypeBeginUnprepareXID
   kTypeWideColumnEntity varstring varstring
   kTypeColumnFamilyWideColumnEntity varint32 varstring varstring
   kTypeNoop
varstring :=
   len: varint32
   data: uint8[len]
```
![](/img/post_img/rocksdb-write-process/fig1.png#pic_center)  

`rep_`的开头放置`fixed64`类型的序列号，第二位是`fixed32`类型的当前记录数，随后是各条操作记录。可以看到每条记录都是以`操作类型 + 操作内容`的形式构成的，且为了更契合Rocksdb的需求，同时为了节省空间，采用了一种变长字符串`varstring`的编码方法。关于`rep_`内各种数据类型的编解码实现在`util/coding.h`和`util/coding.cc`内，在本文不做分析。

### `WriteBatch`调用`Put`
``` C++
Status WriteBatch::Put(ColumnFamilyHandle* column_family, const Slice& key,
                       const Slice& value) {
  size_t ts_sz = 0;
  uint32_t cf_id = 0;
  Status s;

  // 利用ColumnFamilyHandle，获取CF的id
  std::tie(s, cf_id, ts_sz) =
      WriteBatchInternal::GetColumnFamilyIdAndTimestampSize(this, column_family);

  if (!s.ok()) {
    return s;
  }

  // 调用WriteBatchInternal::Put开始写入操作记录
  if (0 == ts_sz) {
    return WriteBatchInternal::Put(this, cf_id, key, value);
  }

  needs_in_place_update_ts_ = true;
  has_key_with_ts_ = true;
  std::string dummy_ts(ts_sz, '\0');
  std::array<Slice, 2> key_with_ts{{key, dummy_ts}};
  return WriteBatchInternal::Put(this, cf_id, SliceParts(key_with_ts.data(), 2),
                                 SliceParts(&value, 1));
}
```
`WatchBatchInternal`是辅助`WatchBatch`的工具类，里面定义了一系列辅助方法。`WriteBatch::Put`先根据列族的Handle获取了列族的Id，随后再调用了`WatchBatchInternal::Put`，送入当前`WatchBatch`指针、列族Id、key和value，开始记录操作。
``` C++
Status WriteBatchInternal::Put(WriteBatch* b, uint32_t column_family_id,
                               const SliceParts& key, const SliceParts& value) {
  Status s = CheckSlicePartsLength(key, value);
  if (!s.ok()) {
    return s;
  }

  // 创建一个保存点，保存当前状态
  LocalSavePoint save(b);

  // 设置WatchBatch->rep_里的count
  WriteBatchInternal::SetCount(b, WriteBatchInternal::Count(b) + 1);

  // 开始记录操作，首先写入操作类型，根据是否为目标列族是否为默认列族设定kTypeValue或kTypeColumnFamilyValue
  if (column_family_id == 0) {
    b->rep_.push_back(static_cast<char>(kTypeValue));
  } else {
    b->rep_.push_back(static_cast<char>(kTypeColumnFamilyValue));
    PutVarint32(&b->rep_, column_family_id);
  }
  // 分别写入key和value
  PutLengthPrefixedSliceParts(&b->rep_, key);
  PutLengthPrefixedSliceParts(&b->rep_, value);
  // 修改WatchBatch的flag，添加HAS_PUT标志表示该batch中有Put操作
  b->content_flags_.store(
      b->content_flags_.load(std::memory_order_relaxed) | ContentFlags::HAS_PUT,
      std::memory_order_relaxed);
  if (b->prot_info_ != nullptr) {
    // See comment in first `WriteBatchInternal::Put()` overload concerning the
    // `ValueType` argument passed to `ProtectKVO()`.
    b->prot_info_->entries_.emplace_back(ProtectionInfo64()
                                             .ProtectKVO(key, value, kTypeValue)
                                             .ProtectC(column_family_id));
  }

  // 提交并解锁WatchBatch
  return save.commit();
}
```
如上注释，`WriteBatchInternal::Put()`会先创建一个保存点，用于保存当前batch的情况。紧接着会对`rep_`的count部分增加计数1，随后添加`Put`记录。最后`save.commit()`会判断`rep_`是否超出了容许的最大范围，若超出则会回退至记录前的状态，并返回一个`Status::MemoryLimit()`信号，而非正常的`Status::OK()`。至此，便`WriteBatch`完成了对`Put`操作的记录。接下来就由`DB::Put`内的`Write(opt, &batch);`语句开始将batch写入数据库。

### 调用`Write`把`WriteBatch`更新写入数据库
在实际使用过程中，由于我们操作的是`DBImpl`对象，所以实际上会使用到`DBImpl::Write()`，它被定义在了`db/db_impl/db_impl_write.cc`里：
``` C++
Status DBImpl::Write(const WriteOptions& write_options, WriteBatch* my_batch) {
  Status s;
  if (write_options.protection_bytes_per_key > 0) {
    s = WriteBatchInternal::UpdateProtectionInfo(
        my_batch, write_options.protection_bytes_per_key);
  }
  if (s.ok()) {
    s = WriteImpl(write_options, my_batch, /*callback=*/nullptr,
                  /*log_used=*/nullptr);
  }
  return s;
}
```
可以看到它还会再进一步调用`DBImpl::WriteImpl()`。`DBImpl::WriteImpl()`代码量非常多，将近500行，包含了许多预处理和不同的写入分支。这个过程中大致可以概括成三个分支：(1)只写入到WAL，会跳转`WriteImplWALOnly()`；(2)Pipeline写入，会跳转`PipelinedWriteImpl()`；(3)非Pipeline写入，继续执行。关注非Pipeline写入，又分为两种情况：是否能够对Memtables多线程并发写入（通过option配置）。在这里我们继续沿着默认的单线程写入，则其关键代码如下：
``` C++
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, WriteCallback* callback,
                         uint64_t* log_used, uint64_t log_ref,
                         bool disable_memtable, uint64_t* seq_used,
                         size_t batch_cnt,
                         PreReleaseCallback* pre_release_callback,
                         PostMemTableCallback* post_memtable_callback) {
  ...
  // 对传入的WriteBatch构建Writer，并将其加入write_thread
  WriteThread::Writer w(write_options, my_batch, callback, log_ref,
                        disable_memtable, batch_cnt, pre_release_callback,
                        post_memtable_callback);
  write_thread_.JoinBatchGroup(&w);
  ...
  {
    ...
    // 在写入前作检查和预处理
    status = PreprocessWrite(write_options, &log_context, &write_context);
    ...
  }
  ...
  if (status.ok()) {
    PERF_TIMER_GUARD(write_memtable_time);
    if (!parallel) {
      // 非并发插入数据
      w.status = WriteBatchInternal::InsertInto(
          write_group, current_sequence, column_family_memtables_.get(),
          &flush_scheduler_, &trim_history_scheduler_,
          write_options.ignore_missing_column_families,
          0 /*recovery_log_number*/, this, parallel, seq_per_batch_,
          batch_per_txn_);
    } else {
      ...
    }
  ...
  }
}
```
这里根据传入的batch对象构建了一个`WriteThread::Writer`对象，并通过`JoinBatchGroup()`将其加入到一个Group中，随后会进行WAL的写入。接着`PreprocessWrite()`会在写入前进行检查和预处理，并可能触发Flush操作。完成上述工作后，`WriteBatchInternal::InsertInto()`会展开真正的写入工作。以上所都是写入流程中的关键操作，但碍于篇幅，本文将仅继续对`WriteBatchInternal::InsertInto()`进行深入。

## `WatchBatch`记录写入数据库
从`WriteBatchInternal::InsertInto()`开始，才可以算是`Put`写入真正开始执行的地方。`WriteBatchInternal::InsertInto()`的实现放在了`db/write_batch.cc`：
``` C++
Status WriteBatchInternal::InsertInto(
    WriteThread::Writer* writer, SequenceNumber sequence,
    ColumnFamilyMemTables* memtables, FlushScheduler* flush_scheduler,
    TrimHistoryScheduler* trim_history_scheduler,
    bool ignore_missing_column_families, uint64_t log_number, DB* db,
    bool concurrent_memtable_writes, bool seq_per_batch, size_t batch_cnt,
    bool batch_per_txn, bool hint_per_batch) {
#ifdef NDEBUG
  (void)batch_cnt;
#endif
  assert(writer->ShouldWriteToMemtable());
  // 构建MemTableInserter
  MemTableInserter inserter(sequence, memtables, flush_scheduler,
                            trim_history_scheduler,
                            ignore_missing_column_families, log_number, db,
                            concurrent_memtable_writes, nullptr /* prot_info */,
                            nullptr /*has_valid_writes*/, seq_per_batch,
                            batch_per_txn, hint_per_batch);
  // 设置LSN
  SetSequence(writer->batch, sequence);
  inserter.set_log_number_ref(writer->log_ref);
  inserter.set_prot_info(writer->batch->prot_info_.get());
  // 开始写入
  Status s = writer->batch->Iterate(&inserter);
  assert(!seq_per_batch || batch_cnt != 0);
  assert(!seq_per_batch || inserter.sequence() - sequence == batch_cnt);
  if (concurrent_memtable_writes) {
    inserter.PostProcess();
  }
  return s;
}
```
函数里根据传入的`writer`和所有其他信息构建了一个`MemTableInserter`，并对`writer->batch`设置了序列号LSN，最后调用`writer->batch->Iterate()`传入之前构建的`inserter`开始写入。`Iterate()`实现如下：
``` C++
Status WriteBatch::Iterate(Handler* handler) const {
  if (rep_.size() < WriteBatchInternal::kHeader) {
    return Status::Corruption("malformed WriteBatch (too small)");
  }

  return WriteBatchInternal::Iterate(this, handler, WriteBatchInternal::kHeader,
                                     rep_.size());
}

Status WriteBatchInternal::Iterate(const WriteBatch* wb,
                                   WriteBatch::Handler* handler, size_t begin,
                                   size_t end) {
  ...
  // 循环不断读取操作记录并执行
  while (((s.ok() && !input.empty()) || UNLIKELY(s.IsTryAgain()))) {
    ...
    if (LIKELY(!s.IsTryAgain())) {
      last_was_try_again = false;
      tag = 0;
      column_family = 0;  // default

      // 从WriteBatch的rep_中解码读取当前的一条记录
      s = ReadRecordFromWriteBatch(&input, &tag, &column_family, &key, &value,
                                   &blob, &xid);
      if (!s.ok()) {
        return s;
      }
    } else {
      ...
    }

    switch (tag) {
      case kTypeColumnFamilyValue:
      case kTypeValue:
        assert(wb->content_flags_.load(std::memory_order_relaxed) &
               (ContentFlags::DEFERRED | ContentFlags::HAS_PUT));
        // 写入数据库
        s = handler->PutCF(column_family, key, value);
        if (LIKELY(s.ok())) {
          empty_batch = false;
          found++;
        }
        break;
      case ...:
        ...
    }
  }

  if (!s.ok()) {
    return s;
  }
  if (handler_continue && whole_batch &&
      found != WriteBatchInternal::Count(wb)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}
```
`Iterate()`会进行循环处理，在每次循环中通过`ReadRecordFromWriteBatch()`解码并读取`WriteBatch::rep_`中最前面的操作记录，获得操作类型`tag`、列族`column_family`以及数据`key`和`value`等信息。随后进入switch块，利用`tag`判断进入不同的分支。对于`Put`操作，会调用`MemTableInserter`的`PutCF()`，送入参数列族和键值对:
``` C++
Status PutCF(uint32_t column_family_id, const Slice& key,
               const Slice& value) override {
    const auto* kv_prot_info = NextProtectionInfo();
    Status ret_status;
    if (kv_prot_info != nullptr) {
      // Memtable needs seqno, doesn't need CF ID
      auto mem_kv_prot_info =
          kv_prot_info->StripC(column_family_id).ProtectS(sequence_);
      ret_status = PutCFImpl(column_family_id, key, value, kTypeValue,
                             &mem_kv_prot_info);
    } else {
      ret_status = PutCFImpl(column_family_id, key, value, kTypeValue,
                             nullptr /* kv_prot_info */);
    }
    // TODO: this assumes that if TryAgain status is returned to the caller,
    // the operation is actually tried again. The proper way to do this is to
    // pass a `try_again` parameter to the operation itself and decrement
    // prot_info_idx_ based on that
    if (UNLIKELY(ret_status.IsTryAgain())) {
      DecrementProtectionInfoIdxForTryAgain();
    }
    return ret_status;
  }
```
可以看到，在其中再调用`PutCFImpl()`，其实现关键源码如下：
``` C++
Status PutCFImpl(uint32_t column_family_id, const Slice& key,
                   const Slice& value, ValueType value_type,
                   const ProtectionInfoKVOS64* kv_prot_info) {
    ···
    Status ret_status;
    // 通过column_family_id找到列族ColumnFamilyData
    if (UNLIKELY(!SeekToColumnFamily(column_family_id, &ret_status))) {
      ...
      return ret_status;
    }
    assert(ret_status.ok());
    
    // 获取当前列族的MemTable
    MemTable* mem = cf_mems_->GetMemTable();
    auto* moptions = mem->GetImmutableMemTableOptions();
    // inplace_update_support is inconsistent with snapshots, and therefore with
    // any kind of transactions including the ones that use seq_per_batch
    assert(!seq_per_batch_ || !moptions->inplace_update_support);
    if (!moptions->inplace_update_support) {
      // 对MemTable写入数据
      ret_status =
          mem->Add(sequence_, value_type, key, value, kv_prot_info,
                   concurrent_memtable_writes_, get_post_process_info(mem),
                   hint_per_batch_ ? &GetHintMap()[mem] : nullptr);
    } else if (moptions->inplace_callback == nullptr ||
               value_type != kTypeValue) {
      ...
    } else {
      ...
    }

    if (UNLIKELY(ret_status.IsTryAgain())) {
      assert(seq_per_batch_);
      const bool kBatchBoundary = true;
      MaybeAdvanceSeq(kBatchBoundary);
    } else if (ret_status.ok()) {
      MaybeAdvanceSeq();
      CheckMemtableFull();  // 检查MemTable是否已满
    }
    // optimize for non-recovery mode
    // If `ret_status` is `TryAgain` then the next (successful) try will add
    // the key to the rebuilding transaction object. If `ret_status` is
    // another non-OK `Status`, then the `rebuilding_trx_` will be thrown
    // away. So we only need to add to it when `ret_status.ok()`.
    if (UNLIKELY(ret_status.ok() && rebuilding_trx_ != nullptr)) {
      assert(!write_after_commit_);
      // TODO(ajkr): propagate `ProtectionInfoKVOS64`.
      ret_status = WriteBatchInternal::Put(rebuilding_trx_, column_family_id,
                                           key, value);
    }
    return ret_status;
  }
```
`PutCFImpl()`中，`SeekToColumnFamily()`会根据`column_family_id`找到对应的列族对象，把指针赋予该`MemTableInserter`的`cf_mems_->current_`变量上，其中`cf_mems_`是`MemTableInserter`的一个`ColumnFamilyMemTablesImpl*`成员变量，`current_`则是`ColumnFamilyMemTablesImpl`的一个`ColumnFamilyData*`成员变量。  
随后，`mem = cf_mems_->GetMemTable()`会得到列族的MemTable指针，再通过`mem->Add()`完成MemTable的写入工作。具体的写入逻辑不再深究，RocksDB为MemTable提供了多种不同的数据结构实现，每一种的写入逻辑都有不同，但都完成了封装提供`Add()`接口以供使用。  
写入完成后，`MaybeAdvanceSeq()`和`CheckMemtableFull()`会进行收尾工作，增大序列号以及检查MemTable是否已满，若已满则会将其转化为ImmuMemTable，相关内容不在本文进行分析。

## 结束
以上就是RocksDB在进行`Put`写入操作时的主要流程和调用关系，至此RocksDB就完成了一个`Put`写入。对于其他的操作也可以参考以上的流程进行逐步深入分析。  
**最后需要再次强调**，RocksDB真正的写入流程非常复杂，本文仅仅是以`Put`操作写入到MemTable为目的对源代码进行追踪，因此分析过程中对许多重要操作一笔带过而对简单的流程分析显得有些啰嗦，例如`WriteImpl()`中不同的分支、`PreprocessWrite()`可能触发的Flush操作，以及`CheckMemtableFull()`对MemTable状态的检查，还有过程中许多重要数据结构的相互引用关系，本文都没有作分析。因此，读者可以把本文视为一篇对RocksDB的分析入门引导，目的在于为读者以及自己今后对其它部分的分析提供一份路线图，帮助理解各种事件发生的时间点，而不是对RocksDB核心技术源码的分析。
