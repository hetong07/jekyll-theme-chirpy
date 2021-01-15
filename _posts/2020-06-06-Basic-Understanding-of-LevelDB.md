---
title: Basic Understanding of LevelDB 
author: hetong07
date: 2020-06-06 23:22:00 -0700
categories: [Blogging, Technical]
tags: [distributed system, leveldb, KV store]
---

LevelDB is the open-sourced Google BigTable implementaion, and its coding style is nice for amateurs to follow. This post presents my understands of the LevelDB and some details of its implementaion. With those components below, one can simulate how LevelDB works in situations, such as, open, read, write, snapshot, failure recovery.

## Table file structure (levels)
The underlying table data structure maintained by the LevelDB (**Google Big Table**) is:   
```
------------------------------------------------   

Memtable (skiplist)   

Immutable memtable (memtable with an immutable marker)   

Level-0 files (persistence of immutable)   

Level-n files   

------------------------------------------------   
```
K-V pairs within each file/table are in order (internal key order), but only files/tables below level-1 (incl.) are disjoint.   

## Log files 
There are two kinds of log files: the MANIFEST file, which persists version information, and the other one is the write/delete log (*.log), which stores the K-V pairs before execution. 

There is only one MANIFEST file for each database open. 

It is noted that the write/delete log has file size limitations, so there may exist many log files during LevelDB runs. To distinguish those write/delete log files, the LevelDB exploits a **log_number**. It is the file number used when creating the log files, and are recorded in a **VersionSet** instance.  

The new log file is created when:   
- Opening a database (`DBImpl::Open()`)   
- Switching to a new memtable (`DBImpl::MakeRoomForWrite()`).   

## Version  
The view of the table file structure is called version, and it is recorded in an instance of **Version** class (defined in *version_edit.h*). Each database contains a bunch of such versions, each time it compacts database. Those versions are managed by a **VersionSet** class, backed by a circular linked list. The **Version** class and its friends **VersionEdit** class and **VersionSet** class share some common variables:  
- Log number, record the log file number;
- Prev log number, record the previous log file number;
- Next file number, record the file number used so far;
- Last sequence number. 
- Manifest_file_number.   
- Compact pointer, store the last compacted key for each level.   

Versions act as checkpoints in LevelDB. Each time the file structure is about to change, i.e., after compaction, LevelDB creates a new version and put it to the head of the version set (`VersionSet::LogAndApply()`). Those changing moments are:   
- Write immutable memtable to L0 files (`DBImpl::CompactMemTable()`);  
- Background compaction (`DBImpl::BackgroundCompaction()`);   
- Install compaction result (`DBImpl::InstallCompactionResults()`);
- Database deletion (`DB::Open()`). 

Not clear why they separate the **Version** and the **VersionEdit** class. The author stated that this would reduce the copy in the comments (*version_set.cc*).  

During the LevelDB recovery (usually when opening a database), it first searches the MANIFEST log file, then applies the recovered records one by one (`VersionSet::Recover()`) into a temporary **VersionEdit** instance. After recovering all previous version info, a new **Version** instance is created and insert the corresponding version set. With the version info, LevelDB searches all files in the database directory to reconstruct file structure. One of the most important things is to recover log files (not the MANIFEST) and reconstruct the memtable.  

It worth noting that there is only one MANIFEST file for each data `Open()` call.   

## Read & Write Interface 
`Get()` is multi-thread compatible. It first acquires the lock to read some shared information and then execute the read without the lock. Since the skiplist can support multi-thread, this could improve the read bandwidth. 

Each writes to LevelDB is wrapped into a **WriteBatch** instance and inserted into a writer list (`DBImpl::Write()`) with a lock. Those **WriteBatch** instances are persisted to the log file (*db_impl.cc:1240*) before executed. To coordinate multi-thread writes, the LevelDB uses a conditional variable to linearize writes. Thus, the write bandwidth cannot benefit from multi-thread. 

## Sequence number   
Each K-V pair added/deleted is assigned with a sequence number for linearization purposes. Consider there are modifications to the same key, levelDB execute the K-V pairs in the sequence number order. 

When searching, the `Get()` automatically uses the snapshot sequence number or use the last sequence number, instead, if not specified.   

The levelDB records the sequence number executed so far in the **Version**/**VersionSet** instance. 

## Other numberings 
Except for the sequence number, there is a file number (variable *new_file_number*). It is monolithically increasing, and each time of creating a new file, the levelDB calls the `NewFileNumber()` to get a file number.   

## Snapshot  
The snapshot subsystem stores the sequence number when a snapshot is made by `leveldb_create_snapshot()` (in file *c.cc*). Snapshots are also stored in a link list. 

I haven't pay much attention to the snapshot.   

## Internal Key 
The internal key for searching (`LookupKey::LookupKey()`) is:   
```
{user_key size + 8} {user_key} {sequence number} {value type} 
```
The value type identifies this is a modification key or a deletion key. This information is used during compaction (`DBImpl::DoCompactionWork()`).   

## Compaction   
The LevelDB uses a dedicated thread (in file *env.h*) as background work for compaction. To initiate a compaction, front threads put compaction jobs to the compaction job queue via the `DBImpl::MaybeScheduleCompaction()` call. 

Major compaction work is done in `DBImpl::BackgroundCompaction()`.    

The compaction information is stored in the **Compaction**/**ManualCompaction** class:   

Files in two adjacent two levels, filled by `Compaction* PickCompaction()`.   

**VersionEdit** instance that holds a set of file for addition and deletion   

Right at the beginning of the compaction, the levelDB creates a table file via the `DBImpl::OpenCompactionOutputFile()`. After such table file size reaches the limit, it uses the `DBImpl::FinishCompactionOutputFile()` to close the file and records that table file name in variable *pending_outpus_*. Then it starts another file creation cycle and continues the compaction.  

When compaction happens, it searches every key in every table in the **Compaction** instance and puts it into output file when 1) if the key is the first time seen, and 2) if the key has a sequence number more significant than specified. Otherwise, it drops the K-V pair. In this way, the compacted file stores a smaller number of K-V pairs.    

After compaction, the `DBImpl::CleanupCompaction()` and `DBImpl::RemoveObsoleteFiles()` are called to do some clean-up work and delete old input files. 

If it is a trivial compaction, the LevelDB skips the above process and only modifies the deletion and addition file set.   

The last step of the compaction is to modify the **VersionEdit** instance this compaction holds via the `DBImpl::InstallCompactionResults()` interface:  
- Mark input files as deleted   
- Mark the generated files as the file to add   

It then calls the `VersionSet::LogAndApply()` to log and apply the changes as we discussed above.  

The compaction results of each level are recorded in different **CompactionState** instances.   

## Reference system   

Many class has a pair of `Ref()`/`Unref()`, to manage its memory, Not clear why, but as far as I know, the C++11 has a builtin support for reference count, and the **Caffe** adopts the build-in solution.   

## IO calculation 
(TODO), and relationship with compaction. 
 
## Other Materials 
There are many materials discusse the internal of the LevelDB in detail. However, I have not read them, thus, cannot guarantee their understandings are the same as mine. But it is worth listing some of them here if you want to gain advanced understanding of the LevelDB, for example, skiplist, memory barrier, etc.

1. [leveldb实现解析](https://yuerblog.cc/wp-content/uploads/leveldb%E5%AE%9E%E7%8E%B0%E8%A7%A3%E6%9E%90.pdf)
2. http://cighao.com/2016/08/13/leveldb-source-analysis-01-introduction/
3. http://woofy.cn/2017/02/13/leveldb1/

## Reference
1. Chang, Fay, et al. "Bigtable: A distributed storage system for structured data." ACM Transactions on Computer Systems (TOCS) 26.2 (2008): 1-26.
2. https://github.com/google/leveldb