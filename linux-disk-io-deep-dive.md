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

- 对于机械磁盘来讲，正如我们之前提到的，随机 IO 自然比顺序 IO 慢是因为它需要更多的磁头寻道与盘片旋转
- 对于固态磁盘来讲，尽管其随机性能比机械磁盘性能更好，它同时有“先擦后写”的限制，随机读写会导致大量的垃圾收集，所以相应地，随机性能依然比连续 I/O 性能差。


In addition, mechanical disks and solid-state disks each have a minimum read and write unit.
The minimum read and write unit of a mechanical disk is a sector, with a general size of 512 bytes.
The minimum read and write unit of the solid-state disk is page, usually, the size is 4KB, 8KB, etc.


除此之外，机械磁盘与固态磁盘各自都拥有最小的读写单元。

- 机械磁盘最小读写单元是一个扇区，一般大小为 512 字节
- 固态磁盘最小的读写单元是一页，通过大小为 4kb, 8kb 等等




However, I also mentioned that it is very inefficient to read and write units as small as 512 bytes each time. Therefore, the file system will form consecutive sectors or pages into logical blocks, and then use logical blocks as the smallest unit to manage data. The size of a common logical block is 4KB, that is, 8 consecutive sectors, or a single page, can form a logical block.



但是，我也提到过每次读写小到 512 字节的单元是非常低效的。 因此，文件系统会将连续的扇区或页面组成逻辑块，然后以逻辑块为最小单位来管理数据。 一个普通逻辑块的大小为4KB，即8个连续的扇区，或者单个页面，可以组成一个逻辑块。


Disk Interface
In addition to classification according to storage media, another common classification method is to classify according to the interface, for example, hard disks can be divided into IDE (Integrated Drive Electronics), SCSI (Small Computer System Interface), SAS (Serial Attached SCSI), SATA (Serial ATA), FC (Fibre Channel), etc.

## Disk Interface

除了按照存储媒介来分类，另外一种常用的分类方式是根据接口。
比如，固态磁盘可以被划分为 IDE（Integrated Drive Electronics）, SCSI(Small Computer System Interface), SAS(Serial Attached SCSI), SATA(Serial ATA), FC(Fibre Channel) 等等。


Different interfaces are often assigned different device names. For example, IDE devices will be assigned an hd prefixed device name, and SCSI and SATA devices will be assigned an sd prefixed device name. If there are multiple disks of the same type, they will be numbered in the alphabetical order of a, b, c, etc.



不同的接口经常被分配不同的设备名称。
比如，IDE 设备分配到以 hd 前缀的设备名称，
SCAI 与 SATA 设备分配以 sd 前缀的设备名称。
如果有多个相同类型的磁盘，它们将按照 a、b、c 等字母顺序进行编号。


In addition to the classification of the disks themselves, when you connect the disks to the server, they can be divided into different architectures according to different usage methods.


除了磁盘本身的分类外，当你将磁盘连接到服务器时，可以根据不同的使用方式将它们划分为不同的架构。

The simplest is to use it directly as an independent disk device. These disks are often divided into different logical partitions as needed, and each partition is numbered. For example, the /dev/sda we used many times before can also be divided into two partitions /dev/sda1 and /dev/sda2.


最简单是将其作为一个独立的磁盘设备来使用，通过会根据需求来将其划分为不同的逻辑分区，并且每个分区被编号。
比如，我们之前经常使用的 /dev/sda 也能够划分为两个分区 /dev/sda1 与 /dev/sda2.


RAID
Another commonly used architecture is to combine multiple disks into a logical disk to form a redundant independent disk array, that is, RAID (Redundant Array of Independent Disks), which can improve the performance of data access and enhance the reliability of data storage.
Depending on the capacity, performance, and reliability requirements, RAID can generally be divided into multiple levels, such as RAID0, RAID1, RAID5, RAID10, and so on.


## RAID

另外一种常用的架构方式是将多个磁盘结合为一个逻辑磁盘来组成一个冗余独立的磁盘阵列。

这就是 RAID（Redundant Array of Independent Disks），这能够用来提升数据访问性能以及加强数据存储的可靠性。

根据容量，性能与可靠性要求，RAID 通常可以划分为多个等级，比如 RAID0, RAID1, RAID5, RAID10 等等。


RAID0 has the best read and write performance, but does not provide data redundancy.
Other levels of RAID, on the basis of providing data redundancy, also have a certain degree of optimization for read and write performance.

- RAID0 拥有最好的读写性能，但是没有提供数据冗余
- 其他等级的 RAID, 在提供数据冗余的基础上，也拥有一定程序上的优化




Major and Minor Device Number
In Linux, the disk is actually managed as a block device, that is, data is read and written in block units, and random read and write is supported. Each block device is assigned two device numbers, the major and minor device numbers. The major device number is used in the driver to distinguish device types; the minor device number is used to number multiple devices of the same type.


## Major and Minor Device Number

在 linux 中，磁盘事实上是作为一个块设备被管理的，这意味着数据是以块单位来读写，并且支持随机读写。

每一个块设备被分配两个设备编号：主次设备号。

主设备号在驱动中被用来区分设备类型
次设备号用于对同一类型的多个设备进行编号。




## Generic Block Layer

Similar to VFS, Linux manages various block devices through a unified general block layer.
The general block layer is actually a block device abstraction layer between the file system and the disk drive. It mainly has two functions.


与 VFS 相似，Linux 通过一个统一的通用块层来管理块设备。


通用块层事实上是处于文件系统与磁盘设备中间的块设备抽象层。

它主要有两个作用。

It provides standard interfaces for accessing block devices for file systems and application, it also abstracts various heterogeneous disk devices into unified block devices, and provides a unified framework to manage the drivers of these decices.
It queues the I/O requests sent by the file system and applications, and improves disk reading/writing efficiency through reordering, request merging, etc.


- 它为文件系统和应用程序提供访问块设备的标准接口，还将各种异构磁盘设备抽象为统一的块设备，并提供统一的框架来管理这些设备的驱动程序。
- 它将文件系统和应用程序发送的 I/O 请求排队，并通过重新排序、请求合并等提高磁盘读/写效率。


Among them, the process of sorting I/O requests is the I/O scheduling that we are familiar with. In fact, the Linux kernel supports four I/O scheduling algorithms, namely NONE, NOOP, CFQ, and DeadLine.

其中，排序 IO 请求的过程就是我们所熟悉的 IO 调度。
事实上，Linux kernel 支持四种 IO 调度算法，NONE, NOOP, CFQ 与 DeadLine.



I/O stack
After understanding the working principles of the disk and the general block layer, and combining them with the file system principle we talked about in the previous issue, we can look at the I/O principle of the Linux storage system as a whole.
We can divide the I/O stack of the Linux storage system into three layers from top to bottom, namely the file system layer, the general block layer and the device layer. The relationship between these three I/O layers is shown in the figure below, which is actually the panorama of the I/O stack of the Linux storage system.

## I/O stack

在明白磁盘与通用块层的工作原理后，并且将它们与文件系统原理结合起来，我们可以整体看看 Linux 存储系统的 I/O 原理。

我们可以由顶至下将 Linux 存储系统的 I/O 栈划分为三层，文件系统层，通用块层以及设备层。

下面的图展示了它们三层单的关系，这实际上是 Linux 存储系统 I/O 栈的全景图。

Based on this panorama of the I/O stack, we can more clearly understand how storage system I/O works.
The file system layer, includes the implementation of the virtual file system and various other file systems. It provides a standard file access interface for upper-layer applications; the lower layer will store and manage disk data through the general block layer.
Generic block layer, including block device I/O queues and I/O schedulers. It queues I/O requests to the file system, reorders and merges requests, and then sends them to the next device layer.
The device layer, including storage devices and corresponding drivers, is responsible for the I/O operations of the final physical device.



基于这张 I/O 栈全景图，我们可以更加清晰地明白存储系统 I/O 是如何工作的。

- 文件系统层，包含了虚拟文件系统的实现以及许多其他的文件系统。它提示一个标准的文件访问接口给上层应用；较低层会通过通用块层来存储以及管理磁盘数据
- 通用块层：包含块设备 I/O 请求以及 I/O 调度。它将 I/O 请求排队到文件系统，重新排序和合并请求，然后将它们发送到下一个设备层。
- 设备层：包含存储设备与对应的驱动，它负责最终物理设备的 I/O 操作。


The I/O of the storage system is usually the slowest link in the entire system. Therefore, Linux optimizes I/O efficiency through various caching mechanisms.
For example, in order to optimize the performance of file access, various caching mechanisms such as page cache, inode cache, and directory entry cache are used to reduce direct calls to lower-level block devices.
Similarly, in order to optimize the access efficiency of the block device, a buffer is used to cache the data of the block device.


存储系统的 I/O 环节通常是整个系统最慢的。
因此， linux 通过大量的缓存机制来优化 I/O 效率，比如 page cache, inode cache 和 directory entry cache 就被用来调用低级别块设备。
同样地，为了优化访问块设备的效率，缓冲区用于缓存块设备的数据。


Conclusion
The Linux storage system I/O stack consists of the file system layer, the general block layer, and the device layer.
Among them, the general block layer is the core of Linux disk I/O. Upwards, it provides a standard interface for accessing block devices for file systems and applications; downwards, it abstracts various heterogeneous disk devices into a unified block device and responds to I/O sent by the file system and applications. /O requests for reordering, request merging, etc., improve the efficiency of disk access.

## Conclusion

Linux存储系统I/O栈由文件系统层、通用块层和设备层组成。

其中，通用块层是核心。

向上，它为文件系统与应用提示了一套标准的接口用来访问块设备。
向下，它抽象了众多异构磁盘设备到为一个统一块设备以及为文件系统与应用的 I/O 请求进行响应，I/O 请求重排序、请求合并等，提高磁盘访问效率。



## 参考

- [1] [Linux — Disk I/O Deep Dive](https://medium.com/dev-genius/linux-disk-i-o-deep-dive-ccd2dd30b98e)