# easy-fs 练习项实现报告 (EXERCISE_IMPLEMENTATION_REPORT)

`easy-fs-T1L4` 主要在文件系统基础上为 [ch6-ch8] 章节提供核心基础，本报告汇总了与 `easy-fs` 相关的功能点及其具体实现细节。

## 1. 块缓存设计 (Block Cache)
- **机制**：实现了简单的块管理器 `BlockCacheManager`。
- **并发控制**：采用 `Arc<Mutex<BlockCache>>` 的组合方式保证在多核环境下的块级读写。
- **LRU 近似**：尽管是简单的引用计数，但也确保了活跃块不会被过载出内存。

## 2. Inode 索引管理
- **文件分配**：实现了 `DiskInode` 的基本索引结构。
- **目录搜索**：`Inode::find` 能够根据文件名在目录项集合中定位文件的 Inode 番号。
- **动态扩容**：`Inode::write_at` 支持在写入超过当前文件大小时动态扩展 Inode 的索引范围。

## 3. 管道 (Pipe) 通信
- **核心逻辑**：在物理磁盘之外额外支持了内存管道。
- **读写一致性**：利用 `Arc<Spinlock<PipeRingBuffer>>` 构建了环形缓冲区，处理 `read` 和 `write`。
- **多端支持**：正确处理了多个 `PipeReader` 和 `PipeWriter` 的共存情况。

## 4. 文件偏移量 (Offset) 记录
- **FileHandle**：在内核层引入 `FileHandle` 时，我们在 `easy-fs-T1L4` 的语义基础上记录了进程对文件的读写偏移量。
- **原子性支持**：保证了并发读取时偏移量的正确更新。

## 5. 硬链接 (Hard Link) 扩展实现
- **任务目标**：允许不同路径的目录项指向同一个 Inode，并维护引用计数（nlink）。
- **关键文件**：`src/layout.rs` / `src/vfs.rs` / `src/efs.rs`。
- **实现细节**：
  - **DiskInode 增强**：在磁盘结构中增加 `nlink` 字段，初始值为 1。
  - **linkat 实现**：在根目录下新增一个目录项，其 Inode 编号指向原有文件的 Inode，并增加对应 `DiskInode` 的 `nlink` 值。
  - **unlinkat 实现**：减少 `DiskInode` 的 `nlink` 值。只有当 `nlink` 归零且没有打开的句柄时，才真正回收位图中的数据块和 Inode 块。
  - **fstat 支持**：通过 `Inode` 接口暴露 `nlink`、`ino` 和 `mode`（文件类型）信息，供内核系统调用 `fstat` 填充 `Stat` 结构体。

## 总结
通过 `easy-fs-T1L4` 的实现，我们成功构建了一个支持：
- 块设备驱动抽象。
- 文件读写与随机访问。
- 目录项遍历。
- 管道流式通信。
- **硬链接的创建、删除与状态查询（linkat, unlinkat, fstat）**。

该实现为 `ch6` 的硬链接编程练习提供了底层数据结构保障，同时也对 `ch7` 的文件扩展应用以及 `ch8` 的并发访问提供了坚实的存储底座。
