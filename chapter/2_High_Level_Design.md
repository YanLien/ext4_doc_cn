# 2. 高层设计

ext4文件系统将存储设备分割成一系列**块组**。为了减少因为碎片化造成的性能问题，块分配器会尽力将每个文件的块保持在同一个组内，从而减少寻道时间。一个块组的大小在超级块的`s_blocks_per_group`字段中指定，以块为单位，尽管也可以通过`8 * block_size_in_bytes`计算得到。以默认的`4KiB`块大小来说，每个组将包含`32,768`个块，长度为`128MiB`。块组的数量是设备大小除以块组大小。

**`ext4`中的所有字段都以小端序写入磁盘。但是，`jbd2`（日志）中的所有字段都以大端序写入磁盘**。

## 2.1. 块

ext4以"块"为单位分配存储空间。一个块是由`1KiB`到`64KiB`之间的扇区组成的组，并且扇区数量必须是2的整数次幂。块又被分组成更大的单位，称为块组。块大小在创建文件系统时指定，通常为`4KiB`。如果块大小大于页面大小，可能会遇到挂载问题（例如，在只有`4KiB`内存页面的i386上使用`64KiB`的块）。默认情况下，一个文件系统最多可以包含2^32个块；如果启用了'64bit'特性，那么文件系统可以拥有2^64个块。结构的位置是以结构所在的块号存储的，而不是磁盘上的绝对偏移量。

对于32位文件系统，限制如下：

| 条目 | 1KiB | 2KiB | 4KiB | 64KiB |
| :------: | ---- | ---- | ---- | ---- |
|块数| 2^32 | 2^32 | 2^32 | 2^32 |
|inode数| 2^32 | 2^32 | 2^32 | 2^32 |
| 文件系统大小 | 4TiB | 8TiB | 16TiB | 256TiB | 
| 每块组块数 | 8,192 | 16,384 | 32,768 | 524,288 |
| 每块组inode数 | 8,192 | 16,384 | 32,768 | 524,288 |
| 块组大小 | 8MiB | 32MiB | 128MiB | 32GiB 
| 每个文件的块数，扩展模式 | 2^32 | 2^32 | 2^32 | 2^32 |
| 每个文件的块数，块映射模式 | 16,843,020 | 134,480,396 | 1,074,791,436 | 4,398,314,962,956 (实际上是2^32，受字段大小限制) |
| 文件大小，扩展模式 | 4TiB | 8TiB | 16TiB | 256TiB |
| 文件大小，块映射模式 | 16GiB | 256GiB | 4TiB | 256TiB |

对于64位文件系统，限制如下：

| 条目 | 1KiB | 2KiB | 4KiB | 64KiB |
| :-------: | ---- | ---- | ---- | ---- |
| 块数 | 2^64 | 2^64 | 2^64 | 2^64 |
| inode数 | 2^32 | 2^32 | 2^32 | 2^32 |
| 文件系统大小 | 16ZiB | 32ZiB | 64ZiB | 1YiB |
| 每块组块数 | 8,192 | 16,384 | 32,768 | 524,288 | 
| 每块组inode数 | 8,192 | 16,384 | 32,768 | 524,288 |
| 块组大小 | 8MiB | 32MiB | 128MiB | 32GiB |
| 每个文件的块数，扩展模式 | 2^32 | 2^32 | 2^32 | 2^32 |
| 每个文件的块数，块映射模式 | 16,843,020 | 134,480,396 | 1,074,791,436 | 4,398,314,962,956 (实际上是2^32，受字段大小限制) |
| 文件大小，扩展模式 | 4TiB | 8TiB | 16TiB | 256TiB |
| 文件大小，块映射模式 | 16GiB | 256GiB | 4TiB | 256TiB |

> 注意：不使用扩展（即使用块映射）的文件必须位于文件系统的前2^32个块内。使用扩展的文件必须位于文件系统的前2^48个块内。对于更大的文件系统会发生什么情况尚不清楚。

## 2.2. 布局

标准块组的布局大致如下（下面将分别讨论这些字段中的每一个）：


|第0组填充|ext4超级块|组描述符|保留的GDT块|数据块位图|inode位图|inode表| 数据块|
| ------ | -------- | ----- | -------- | -------- | ------ | ----- | ---- | 
|1024字节|   1个块  | 多个块 |  多个块  |  1个块  |  1个块  | 多个块 | 更多块|


对于块组0的特殊情况，前1024字节未使用，以允许安装x86引导扇区和其他特殊情况。超级块将从偏移1024字节开始，无论是哪个块（通常是0）。然而，如果由于某种原因块大小 = 1024，那么块0将被标记为使用中，超级块将放在块1中。对于所有其他块组，没有填充。

> 注：无论哪种情况，块0都会被标记为已使用。

ext4驱动程序主要处理块组0中的超级块和组描述符。超级块和组描述符的冗余副本会写入磁盘上的一些块组中，以防磁盘开始部分损坏，尽管并非所有块组都必须托管冗余副本（详见下段）。如果组没有冗余副本，则块组从数据块位图开始。还要注意，当文件系统刚刚格式化时，mkfs将在块组描述符之后和块位图开始之前分配"保留GDT块"空间，以允许将来扩展文件系统。默认情况下，文件系统允许比原始文件系统大小增加1024倍。

The ext4 driver primarily works with the superblock and the group descriptors that are found in block group 0. Redundant copies of the superblock and group descriptors are written to some of the block groups across the disk in case the beginning of the disk gets trashed, though not all block groups necessarily host a redundant copy (see following paragraph for more details). If the group does not have a redundant copy, the block group begins with the data block bitmap. Note also that when the filesystem is freshly formatted, mkfs will allocate “reserve GDT block” space after the block group descriptors and before the start of the block bitmaps to allow for future expansion of the filesystem. By default, a filesystem is allowed to increase in size by a factor of 1024x over the original filesystem size.

inode表的位置由grp.bg_inode_table_*给出。它是一个连续的块范围，足够大以包含sb.s_inodes_per_group * sb.s_inode_size字节。

The location of the inode table is given by grp.bg_inode_table_*. It is continuous range of blocks large enough to contain sb.s_inodes_per_group * sb.s_inode_size bytes.

至于块组中项目的顺序，通常确定超级块和组描述符表（如果存在）将位于块组的开头。位图和inode表可以在任何地方，并且位图可能位于inode表之后，或者两者可能位于不同的组（flex_bg）。剩余空间用于文件数据块、间接块映射、扩展树块和扩展属性。

As for the ordering of items in a block group, it is generally established that the super block and the group descriptor table, if present, will be at the beginning of the block group. The bitmaps and the inode table can be anywhere, and it is quite possible for the bitmaps to come after the inode table, or for both to be in different groups (flex_bg). Leftover space is used for file data blocks, indirect block maps, extent tree blocks, and extended attributes.

2.3. Flexible Block Groups
Starting in ext4, there is a new feature called flexible block groups (flex_bg). In a flex_bg, several block groups are tied together as one logical block group; the bitmap spaces and the inode table space in the first block group of the flex_bg are expanded to include the bitmaps and inode tables of all other block groups in the flex_bg. For example, if the flex_bg size is 4, then group 0 will contain (in order) the superblock, group descriptors, data block bitmaps for groups 0-3, inode bitmaps for groups 0-3, inode tables for groups 0-3, and the remaining space in group 0 is for file data. The effect of this is to group the block group metadata close together for faster loading, and to enable large files to be continuous on disk. Backup copies of the superblock and group descriptors are always at the beginning of block groups, even if flex_bg is enabled. The number of block groups that make up a flex_bg is given by 2 ^ sb.s_log_groups_per_flex.

2.4. Meta Block Groups
Without the option META_BG, for safety concerns, all block group descriptors copies are kept in the first block group. Given the default 128MiB(2^27 bytes) block group size and 64-byte group descriptors, ext4 can have at most 2^27/64 = 2^21 block groups. This limits the entire filesystem size to 2^21 * 2^27 = 2^48bytes or 256TiB.

The solution to this problem is to use the metablock group feature (META_BG), which is already in ext3 for all 2.6 releases. With the META_BG feature, ext4 filesystems are partitioned into many metablock groups. Each metablock group is a cluster of block groups whose group descriptor structures can be stored in a single disk block. For ext4 filesystems with 4 KB block size, a single metablock group partition includes 64 block groups, or 8 GiB of disk space. The metablock group feature moves the location of the group descriptors from the congested first block group of the whole filesystem into the first group of each metablock group itself. The backups are in the second and last group of each metablock group. This increases the 2^21 maximum block groups limit to the hard limit 2^32, allowing support for a 512PiB filesystem.

The change in the filesystem format replaces the current scheme where the superblock is followed by a variable-length set of block group descriptors. Instead, the superblock and a single block group descriptor block is placed at the beginning of the first, second, and last block groups in a meta-block group. A meta-block group is a collection of block groups which can be described by a single block group descriptor block. Since the size of the block group descriptor structure is 64 bytes, a meta-block group contains 16 block groups for filesystems with a 1KB block size, and 64 block groups for filesystems with a 4KB blocksize. Filesystems can either be created using this new block group descriptor layout, or existing filesystems can be resized on-line, and the field s_first_meta_bg in the superblock will indicate the first block group using this new layout.

Please see an important note about BLOCK_UNINIT in the section about block and inode bitmaps.

2.5. Lazy Block Group Initialization
A new feature for ext4 are three block group descriptor flags that enable mkfs to skip initializing other parts of the block group metadata. Specifically, the INODE_UNINIT and BLOCK_UNINIT flags mean that the inode and block bitmaps for that group can be calculated and therefore the on-disk bitmap blocks are not initialized. This is generally the case for an empty block group or a block group containing only fixed-location block group metadata. The INODE_ZEROED flag means that the inode table has been initialized; mkfs will unset this flag and rely on the kernel to initialize the inode tables in the background.

By not writing zeroes to the bitmaps and inode table, mkfs time is reduced considerably. Note the feature flag is RO_COMPAT_GDT_CSUM, but the dumpe2fs output prints this as “uninit_bg”. They are the same thing.

2.6. Special inodes
ext4 reserves some inode for special features, as follows:

inode Number

Purpose

0

Doesn’t exist; there is no inode 0.

1

List of defective blocks.

2

Root directory.

3

User quota.

4

Group quota.

5

Boot loader.

6

Undelete directory.

7

Reserved group descriptors inode. (“resize inode”)

8

Journal inode.

9

The “exclude” inode, for snapshots(?)

10

Replica inode, used for some non-upstream feature?

11

Traditional first non-reserved inode. Usually this is the lost+found directory. See s_first_ino in the superblock.

Note that there are also some inodes allocated from non-reserved inode numbers for other filesystem features which are not referenced from standard directory hierarchy. These are generally reference from the superblock. They are:

Superblock field

Description

s_lpf_ino

Inode number of lost+found directory.

s_prj_quota_inum

Inode number of quota file tracking project quotas

s_orphan_file_inum

Inode number of file tracking orphan inodes.

2.7. Block and Inode Allocation Policy
ext4 recognizes (better than ext3, anyway) that data locality is generally a desirably quality of a filesystem. On a spinning disk, keeping related blocks near each other reduces the amount of movement that the head actuator and disk must perform to access a data block, thus speeding up disk IO. On an SSD there of course are no moving parts, but locality can increase the size of each transfer request while reducing the total number of requests. This locality may also have the effect of concentrating writes on a single erase block, which can speed up file rewrites significantly. Therefore, it is useful to reduce fragmentation whenever possible.

The first tool that ext4 uses to combat fragmentation is the multi-block allocator. When a file is first created, the block allocator speculatively allocates 8KiB of disk space to the file on the assumption that the space will get written soon. When the file is closed, the unused speculative allocations are of course freed, but if the speculation is correct (typically the case for full writes of small files) then the file data gets written out in a single multi-block extent. A second related trick that ext4 uses is delayed allocation. Under this scheme, when a file needs more blocks to absorb file writes, the filesystem defers deciding the exact placement on the disk until all the dirty buffers are being written out to disk. By not committing to a particular placement until it’s absolutely necessary (the commit timeout is hit, or sync() is called, or the kernel runs out of memory), the hope is that the filesystem can make better location decisions.

The third trick that ext4 (and ext3) uses is that it tries to keep a file’s data blocks in the same block group as its inode. This cuts down on the seek penalty when the filesystem first has to read a file’s inode to learn where the file’s data blocks live and then seek over to the file’s data blocks to begin I/O operations.

The fourth trick is that all the inodes in a directory are placed in the same block group as the directory, when feasible. The working assumption here is that all the files in a directory might be related, therefore it is useful to try to keep them all together.

The fifth trick is that the disk volume is cut up into 128MB block groups; these mini-containers are used as outlined above to try to maintain data locality. However, there is a deliberate quirk -- when a directory is created in the root directory, the inode allocator scans the block groups and puts that directory into the least heavily loaded block group that it can find. This encourages directories to spread out over a disk; as the top-level directory/file blobs fill up one block group, the allocators simply move on to the next block group. Allegedly this scheme evens out the loading on the block groups, though the author suspects that the directories which are so unlucky as to land towards the end of a spinning drive get a raw deal performance-wise.

Of course if all of these mechanisms fail, one can always use e4defrag to defragment files.

2.8. Checksums
Starting in early 2012, metadata checksums were added to all major ext4 and jbd2 data structures. The associated feature flag is metadata_csum. The desired checksum algorithm is indicated in the superblock, though as of October 2012 the only supported algorithm is crc32c. Some data structures did not have space to fit a full 32-bit checksum, so only the lower 16 bits are stored. Enabling the 64bit feature increases the data structure size so that full 32-bit checksums can be stored for many data structures. However, existing 32-bit filesystems cannot be extended to enable 64bit mode, at least not without the experimental resize2fs patches to do so.

Existing filesystems can have checksumming added by running tune2fs -O metadata_csum against the underlying device. If tune2fs encounters directory blocks that lack sufficient empty space to add a checksum, it will request that you run e2fsck -D to have the directories rebuilt with checksums. This has the added benefit of removing slack space from the directory files and rebalancing the htree indexes. If you _ignore_ this step, your directories will not be protected by a checksum!

The following table describes the data elements that go into each type of checksum. The checksum function is whatever the superblock describes (crc32c as of October 2013) unless noted otherwise.

Metadata

Length

Ingredients

Superblock

__le32

The entire superblock up to the checksum field. The UUID lives inside the superblock.

MMP

__le32

UUID + the entire MMP block up to the checksum field.

Extended Attributes

__le32

UUID + the entire extended attribute block. The checksum field is set to zero.

Directory Entries

__le32

UUID + inode number + inode generation + the directory block up to the fake entry enclosing the checksum field.

HTREE Nodes

__le32

UUID + inode number + inode generation + all valid extents + HTREE tail. The checksum field is set to zero.

Extents

__le32

UUID + inode number + inode generation + the entire extent block up to the checksum field.

Bitmaps

__le32 or __le16

UUID + the entire bitmap. Checksums are stored in the group descriptor, and truncated if the group descriptor size is 32 bytes (i.e. ^64bit)

Inodes

__le32

UUID + inode number + inode generation + the entire inode. The checksum field is set to zero. Each inode has its own checksum.

Group Descriptors

__le16

If metadata_csum, then UUID + group number + the entire descriptor; else if gdt_csum, then crc16(UUID + group number + the entire descriptor). In all cases, only the lower 16 bits are stored.

2.9. Bigalloc
At the moment, the default size of a block is 4KiB, which is a commonly supported page size on most MMU-capable hardware. This is fortunate, as ext4 code is not prepared to handle the case where the block size exceeds the page size. However, for a filesystem of mostly huge files, it is desirable to be able to allocate disk blocks in units of multiple blocks to reduce both fragmentation and metadata overhead. The bigalloc feature provides exactly this ability.

The bigalloc feature (EXT4_FEATURE_RO_COMPAT_BIGALLOC) changes ext4 to use clustered allocation, so that each bit in the ext4 block allocation bitmap addresses a power of two number of blocks. For example, if the file system is mainly going to be storing large files in the 4-32 megabyte range, it might make sense to set a cluster size of 1 megabyte. This means that each bit in the block allocation bitmap now addresses 256 4k blocks. This shrinks the total size of the block allocation bitmaps for a 2T file system from 64 megabytes to 256 kilobytes. It also means that a block group addresses 32 gigabytes instead of 128 megabytes, also shrinking the amount of file system overhead for metadata.

The administrator can set a block cluster size at mkfs time (which is stored in the s_log_cluster_size field in the superblock); from then on, the block bitmaps track clusters, not individual blocks. This means that block groups can be several gigabytes in size (instead of just 128MiB); however, the minimum allocation unit becomes a cluster, not a block, even for directories. TaoBao had a patchset to extend the “use units of clusters instead of blocks” to the extent tree, though it is not clear where those patches went-- they eventually morphed into “extent tree v2” but that code has not landed as of May 2015.

2.10. Inline Data
The inline data feature was designed to handle the case that a file’s data is so tiny that it readily fits inside the inode, which (theoretically) reduces disk block consumption and reduces seeks. If the file is smaller than 60 bytes, then the data are stored inline in inode.i_block. If the rest of the file would fit inside the extended attribute space, then it might be found as an extended attribute “system.data” within the inode body (“ibody EA”). This of course constrains the amount of extended attributes one can attach to an inode. If the data size increases beyond i_block + ibody EA, a regular block is allocated and the contents moved to that block.

Pending a change to compact the extended attribute key used to store inline data, one ought to be able to store 160 bytes of data in a 256-byte inode (as of June 2015, when i_extra_isize is 28). Prior to that, the limit was 156 bytes due to inefficient use of inode space.

The inline data feature requires the presence of an extended attribute for “system.data”, even if the attribute value is zero length.

2.10.1. Inline Directories
The first four bytes of i_block are the inode number of the parent directory. Following that is a 56-byte space for an array of directory entries; see struct ext4_dir_entry. If there is a “system.data” attribute in the inode body, the EA value is an array of struct ext4_dir_entry as well. Note that for inline directories, the i_block and EA space are treated as separate dirent blocks; directory entries cannot span the two.

Inline directory entries are not checksummed, as the inode checksum should protect all inline data contents.

2.11. Large Extended Attribute Values
To enable ext4 to store extended attribute values that do not fit in the inode or in the single extended attribute block attached to an inode, the EA_INODE feature allows us to store the value in the data blocks of a regular file inode. This “EA inode” is linked only from the extended attribute name index and must not appear in a directory entry. The inode’s i_atime field is used to store a checksum of the xattr value; and i_ctime/i_version store a 64-bit reference count, which enables sharing of large xattr values between multiple owning inodes. For backward compatibility with older versions of this feature, the i_mtime/i_generation may store a back-reference to the inode number and i_generation of the one owning inode (in cases where the EA inode is not referenced by multiple inodes) to verify that the EA inode is the correct one being accessed.

2.12. Verity files
ext4 supports fs-verity, which is a filesystem feature that provides Merkle tree based hashing for individual readonly files. Most of fs-verity is common to all filesystems that support it; see Documentation/filesystems/fsverity.rst for the fs-verity documentation. However, the on-disk layout of the verity metadata is filesystem-specific. On ext4, the verity metadata is stored after the end of the file data itself, in the following format:

Zero-padding to the next 65536-byte boundary. This padding need not actually be allocated on-disk, i.e. it may be a hole.

The Merkle tree, as documented in Documentation/filesystems/fsverity.rst, with the tree levels stored in order from root to leaf, and the tree blocks within each level stored in their natural order.

Zero-padding to the next filesystem block boundary.

The verity descriptor, as documented in Documentation/filesystems/fsverity.rst, with optionally appended signature blob.

Zero-padding to the next offset that is 4 bytes before a filesystem block boundary.

The size of the verity descriptor in bytes, as a 4-byte little endian integer.

Verity inodes have EXT4_VERITY_FL set, and they must use extents, i.e. EXT4_EXTENTS_FL must be set and EXT4_INLINE_DATA_FL must be clear. They can have EXT4_ENCRYPT_FL set, in which case the verity metadata is encrypted as well as the data itself.

Verity files cannot have blocks allocated past the end of the verity metadata.

Verity and DAX are not compatible and attempts to set both of these flags on a file will fail.