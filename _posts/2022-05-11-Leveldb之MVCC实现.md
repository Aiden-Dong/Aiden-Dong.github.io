---
layout:     post
title:      Leveldb MVCC实现细节 | leveldb 篇
subtitle:   
date:       2022-05-19
author:     Aiden
header-img: img/post-bg-mma-1.jpg
catalog: true  
tags:
    - olap
---

![image.png]({{ site.url }}/assets/leveldb_4_1.jpg)

leveldb 使用`VersionSet`来维护SST的变更记录，SST的变更主要发生在**Compaction**环节，分为**Minor compaction**与**Major compaction**。

**Minor compaction**发生在`MemTable`数据容量超过限制溢出到SST。 **Manor compaction**发生在sst达到了触发合并条件，数据写出到下一层level.

`VersionSet` 里面维护有一个`Version`双向链表。 来记录每一个版本中的有效SST. 当SST发生变更时，会构造一个`VersionEdit`来维护SST变更记录。

在逻辑上来说 : `New Version = Current Version + VersinEdit`

#### Version 的数据结构

```cpp
VersionSet* vset_;  // 表示这个 verset 隶属于哪一个 verset_set, 在 leveldb 中只有一个 versetset

// 历史版本的双向链表
Version* next_;     // Next version in linked list
Version* prev_;     // Previous version in linked list

int refs_;          // 有多少服务还引用这个版本

/***
 * 当前版本的所有数据 -- 二级指针结构
 * 第一层代表每一个 level 级别
 * 第二层代表同一个 level 级别下面的 sst 文件number
 */
std::vector<FileMetaData*> files_[config::kNumLevels];

// 用于压缩的标记
// 压缩触发条件 1 ： 基于文件 seek 的压缩方式
FileMetaData* file_to_compact_;                           // 用于 seek 次数超过阈值之后需要压缩的文件
int file_to_compact_level_;                               // 用于 seek 次数超过阈值之后需要压缩的文件所在的level

// 压缩触发条件 2 ： 基于文件大小超过阈值的压缩方式
double compaction_score_;                                 // 用于检查 size 超过阈值之后需要压缩的文件
int compaction_level_;                                    // 用于检查 size 查过阈值之后需要压缩的文件所在的 level
```

主要包含: 

1. 版本的维护链表
2. 当前有效的SST
3. compaction条件

#### VersionEdit 的数据结构

```cpp
typedef std::set<std::pair<int, uint64_t>> DeletedFileSet;

std::string comparator_;            // 比较器名字
uint64_t log_number_;               // 日志编号， 该日志之前的数据均可删除
uint64_t prev_log_number_;          // 已经弃用
uint64_t next_file_number_;         // 下一个文件编号(ldb, idb, MANIFEST文件共享一个序号空间)
SequenceNumber last_sequence_;      // 最后的 Seq_num

bool has_comparator_;               //
bool has_log_number_;
bool has_prev_log_number_;
bool has_next_file_number_;
bool has_last_sequence_;

std::vector<std::pair<int, InternalKey>> compact_pointers_;   // 存储在本次执行的压缩操作对应的level与压缩的最大的key
DeletedFileSet deleted_files_;                                // 相比上次 version 而言， 本次需要删除的文件有哪些
std::vector<std::pair<int, FileMetaData>> new_files_;         // 相比于上次 version 而言， 本次新增文件有哪些
```

主要包含 : 

1. 当前新增的SST集合
2. 当前删除的SST集合
3. 最新的key SequenceNumber
4. 记录每个level当前合并的key位置



