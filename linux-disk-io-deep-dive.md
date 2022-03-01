First, let’s recap briefly how the Linux file system works. A file system is a mechanism for organizing and managing files on a storage device. In the implementation of the various file systems, Linux abstracts a layer of virtual file system VFS, which defines a set of data structures and standard interfaces that all file systems support.




首先，让我们简要地回顾下 Linux 文件系统工作原理。

文件系统是一种用来组织与管理存储设备上文件的机制。

在众多文件系统的实现中，Linux 抽象出虚拟文件系统（VFS virual file system）层, VFS 定义了一整套所有文件系统都支持的数据结构与标准接口。



In this way, the application only needs to interact with the unified interface provided by VFS, and does not need to pay attention to the specific implementation of the underline file system. Internally VFS manages files through data structures such as directory entries, inodes, logical blocks and superblocks.


按照这种方式，应用只需要与 VFS 提供的统一接口进行交互，而不需要关注于底层文件系统的特定实现。
VFS 内部里通过数据结构如 directory entries, inodes, logical blocks 与 superblocks 来管理文件。

Directory entry, which records the name of the file and the directory relationship between the file and other directory entries. Dentry is a memory cache.
Inode, which records the metadata of the file.
Logical block, the smallest read-write unit composed of consecutive disk sectors, is used to store file data.
Superblock, record the overall status of the file system, such as usage of inodes and logical blocks.

- Directory entry：记录文件的名称以及文件与其他 directory entries 的关系。Directory entery 是内存缓存。
- Inode: 记录了文件的元数据
- Logical block: 由连续的磁盘扇区组成的最小读写单元，用于存储文件数据。
- Superblock： 记录了全面的文件系统状态信息，如 inode 与 logical block 的使用。


So further thinking, how does the disk work? What indicators can be used to measure the disk performance?

所以进一步思考，它是如何工作的？哪些指标可以用来监控磁盘的性能？



## Disk

Disks are devices that can be persistently stored. According to different storage media, common disks can be divided into two categories: mechanical disks and solid-state disks.

