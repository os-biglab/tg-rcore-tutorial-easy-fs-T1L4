# easy-fs 架构设计概览 (ARCHITECTURE_OVERVIEW)

`easy-fs-T1L4` 是一个为教育目的设计的简易文件系统，旨在提供基础的持久化存储能力。相比于传统的 Unix 文件系统，它极大地简化了磁盘布局与并发控制。

## 设计哲学
1. **层次化设计**：从最底层的块设备抽象到最高层的文件/管道接口，每一层职责单一。
2. **块缓存驱动**：所有磁盘读写都经过块缓存（Block Cache），利用引用计数确保并发安全与一致性。
3. **内存即磁盘**：充分利用 Rust 的 `Arc` 和 `Spinlock` 实现简单的内存中元数据管理。

## 核心架构层次

### 1. 块设备抽象 (Device Layer)
定义了 `BlockDevice` trait，任何实现了该接口的结构（如 VirtIO 驱动、或者是内存模拟磁盘）均可挂载 `easy-fs`。
- **关键定义**：位于 `src/lib.rs` 的 `BlockDevice`。

### 2. 块缓存层 (Block Cache Layer)
实现了简单的内存缓存，避免频繁的磁盘寻道。
- **管理器**：`BlockCacheManager`。
- **对象**：`BlockCache`。每一个块在内存中由 `Arc<Spinlock<..>>` 保护，确保在多线程环境下对同一磁盘块的访问是互斥的。

### 3. 磁盘布局层 (On-disk Layout)
定义了位图（Bitmap）、超级块（SuperBlock）、磁盘索引节点（DiskInode）和目录项（DirEntry）。
- **位图**：管理数据块和 Inode 块的分配与回收。
- **Inode**：采用多级索引结构记录文件内容所在的磁盘块。

### 4. 文件系统逻辑 (Filesystem Logic)
核心结构体 `EasyFileSystem`。
- **职责**：初始化磁盘格式（`create`）、打开现有文件系统（`open`）、分配 Inode 和数据块。
- **挂载点**：它持有根 Inode 的引用。

### 5. 高层接口 (Interface Layer)
面向内核提供的抽象接口。
- **Inode**：封装了磁盘 Inode 的高层对象，支持 `ls`、`create`、`read`、`write` 等操作。
- **管道 (Pipe)**：利用共享缓冲区实现的进程间通信（IPC）机制。
- **文件描述符**：统一了文件与管道的底层操作习惯。

## 关键模块关系
- `src/lib.rs`: 导出公共接口。
- `src/vfs.rs`: 虚拟文件系统抽象，定义 `Inode` 及其对底层 `DiskInode` 的操作逻辑。
- `src/efs.rs`: `EasyFileSystem` 的具体管理实现。
- `src/layout.rs`: 磁盘数据结构的内存定义。
- `src/block_cache.rs`: 块缓存实现细节。
- `src/bitmap.rs`: 位图索引与空间管理。
- `src/pipe.rs`: 管道机制的核心实现。
