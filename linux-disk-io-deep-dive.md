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

磁盘是可以用来做永久存储的设备。

根据不同的存储媒介，常用的磁盘可以被分为两类，机械磁盘（mechanical disk）与固态磁盘（solid-state disk）.

Mechanical disks are usually abbreviated as HDD and are mainly composed of a platter and read-write head, and the data is stored in the ring track of the platter. Before reading and writing data, it is necessary to move the read-write head to locate the track where the data is stored, and then access the data.


机械磁盘通常缩写为 HDD，并且主要是由盘片与读写头组成的，数据存储在盘片的环形磁道上。

在读写数据前，需要移动读写头定位到数据存储的磁道然后进行数据访问。


Solid-state disk usually abbreviated as SSD consists of solid-state electronic components. Solid-state disks do not require track addressing, so both sequential I/O and random I/O performance are much better.

固态磁盘通过编写为 SSD。

固态磁盘不需要磁道寻址，因此无论是顺序 IO 还是随机 IO 性能都好得多。

In fact, no matter whether it is a mechanical disk or a solid-state disk, the random I/O of the same disk is much slower than the sequential I/O, and the reason is also obvious.

For mechanical disks, as we just mentioned, random I/O is naturally slower than sequential I/O because it requires more head seeks and platter rotations.
For solid-state disks, although its random performance is much better than that of mechanical hard disks, it also has the limitation of “erasing before writing”. Random read and write will lead to a lot of garbage collection, so correspondingly, the performance of random I/O is still much worse than that of continuous I/O.


事实上，不管是机械磁盘还是固态磁盘，同个磁盘的随机 IO 都会比顺序 IO 慢得多，原因也是显而易见的。

- 对于机械磁盘来讲，正如我们之前提到的，随机 IO 自然比顺序 IO 慢是因为基