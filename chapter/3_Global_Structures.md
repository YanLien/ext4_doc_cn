# 3. 全局结构

文件系统被分割成多个块组，每个块组在固定位置都有静态元数据。

## 3.1. 超级块

超级块记录了有关包含文件系统的各种信息，如块计数、inode计数、支持的功能、维护信息等。

如果设置了`sparse_super`功能标志，那么超级块和组描述符的冗余副本仅保存在组号为0或3、5、7的幂的组中。如果未设置该标志，则冗余副本保存在所有组中。

超级块校验和是针对超级块结构计算的，其中包括文件系统UUID。

ext4超级块在`struct ext4_super_block`中的布局如下：

| 偏移量 | 大小 | 名称 | 描述 |
| ---- | --- | ---- | --- |
| 0x0 | `__le32` | `s_inodes_countinode` | 总数。| 
| 0x4 | `__le32` | `s_blocks_count_lo` | 块总数。|
| 0x8 | __le32 | `s_r_blocks_count_lo` | 这些块只能由超级用户分配。|
| 0xC | __le32 | `s_free_blocks_count_lo` | 空闲块计数。|
| 0x10 | __le32 | `s_free_inodes_count` | 空闲inode计数。|
| 0x14 | __le32 | s_first_data_block | 第一个数据块。对于1k块文件系统，这必须至少为1，对于所有其他块大小，通常为0。|
| 0x18 | __le32 | s_log_block_size | 块大小为`2 ^ (10 + s_log_block_size)`。|
| 0x1C | __le32 | s_log_cluster_size | 如果启用了`bigalloc`，集群大小为`2 ^ (10 + s_log_cluster_size)`块。否则`s_log_cluster_size` | 必须等于s_log_block_size。|
| 0x20 | __le32 | s_blocks_per_group | 每组块数。|
| 0x24 | __le32 | s_clusters_per_group | 如果启用了`bigalloc`，则每组集群数。否则`s_clusters_per_group`必须等于`s_blocks_per_group`。|
| 0x28 | __le32 | s_inodes_per_group | 每组inode数。0x2C__le32s_mtime挂载时间，从纪元开始的秒数。|
| 0x30 | __le32 | s_wtime | 写入时间，从纪元开始的秒数。0x34__le16s_mnt_count自上次fsck以来的挂载次数。|
| 0x36 | __le16 | s_max_mnt_count | 超过此挂载次数需要进行fsck。|
| 0x38 | __le16 | s_magic | 魔术签名，0xEF53 |
| 0x3A | __le16 | s_state | 文件系统状态。更多信息请参见`super_state`。|
| 0x3C | __le16 | s_errors | 检测到错误时的行为。更多信息请参见super_errors。|
| 0x3E | __le16 | s_minor_rev_level | 次要修订级别。|
| 0x40 | __le32 | s_lastcheck | 上次检查时间，从纪元开始的秒数。|
| 0x44 | __le32 | s_checkinterval | 检查之间的最大时间，以秒为单位。|
| 0x48 | __le32 | s_creator_os | 创建操作系统。更多信息请参见super_creator表。|
| 0x4C | __le32 | s_rev_level | 修订级别。更多信息请参见super_revision表。|
| 0x50 | __le16 | s_def_resuid | 保留块的默认uid。0x52__le16s_def_resgid保留块的默认gid。|
|      |      |  | 这些字段仅适用于`EXT4_DYNAMIC_REV`超级块。<br>注意：兼容功能集和不兼容功能集之间的区别在于，如果内核不知道不兼容功能集中设置的位，它应该拒绝挂载文件系统。<br>  注：当系统挂载一个ext文件系统时，它会检查超级块中的功能集标志。如果发现兼容功能集中有不认识的功能，系统会忽略这些功能，但仍然挂载文件系统；而如果发现不兼容功能集中有不认识的功能，系统会拒绝挂载。 <br>`e2fsck`的要求更严格；如果它不了解兼容或不兼容功能集中的功能，它必须中止，不要尝试干预它不理解的东西... <br>|
| 0x54 | __le32 | `s_first_ino` | 第一个非保留inode。|
| 0x58 | __le16 | `s_inode_size` | inode结构的大小，以字节为单位。|
| 0x5A | __le16 | `s_block_group_nr` | 此超级块的块组#。|
| 0x5C | `__le32` | `s_feature_compat` | 兼容功能集标志。即使内核不理解标志，它仍然可以读/写这个文件系统；fsck不应该这样做。更多信息请参见`super_compat`表。|
| 0x60 | `__le32` | s_feature_incompat | 不兼容功能集。如果内核或fsck不理解这些位中的一个，它应该停止。更多信息请参见`super_incompat`表。|
| 0x64 | `__le32` | s_feature_ro_compat | 只读兼容功能集。如果内核不理解这些位中的一个，它仍然可以以只读方式挂载。更多信息请参见super_rocompat表。|
| 0x68 | `__u8` | s_uuid[16] | 卷的128位UUID。 |
| 0x78 | `char` | s_volume_name[16] | 卷标。 |
| 0x88 | `char` | s_last_mounted[64] | 文件系统上次挂载的目录。 |
| 0xC8 | `__le32` | s_algorithm_usage_bitmap | 用于压缩（在`e2fsprogs/Linux`中未使用） |
|  |  |  | 性能提示。目录预分配只有在`EXT4_FEATURE_COMPAT_DIR_PREALLOC`标志打开时才会发生。|
| 0xCC | `__u8` | `s_prealloc_blocks` | 尝试为...文件预分配的块数？（在e2fsprogs/Linux中未使用） |
| 0xCD |` __u8` | `s_prealloc_dir_blocks` | 为目录预分配的块数。（在e2fsprogs/Linux中未使用） |
| 0xCE | `__le16` | `s_reserved_gdt_blocks` | 为未来文件系统扩展保留的GDT条目数。 |
|  |  |  | 仅当设置了`EXT4_FEATURE_COMPAT_HAS_JOURNAL`时，日志支持才有效。|
| 0xD0 | __u8 | s_journal_uuid[16] | 日志超级块的UUID |
| 0xE0 | __le32 | s_journal_inum | 日志文件的inode号。 |
| 0xE4 | __le32 | s_journal_dev | 如果设置了外部日志功能标志，则为日志文件的设备号。 |
| 0xE8 | __le32 | s_last_orphan | 要删除的孤立inode列表的开始。 |
| 0xEC | __le32 | s_hash_seed[4] | HTREE哈希种子。 |
| 0xFC | __u8 | s_def_hash_version | 用于目录哈希的默认哈希算法。更多信息请参见super_def_hash。 |
| 0xFD | __u8 | s_jnl_backup_type | 如果此值为0或EXT3_JNL_BACKUP_BLOCKS (1)，则s_jnl_blocks字段包含inode的i_block[]数组和i_size的副本。 |
| 0xFE | __le16 | s_desc_size | 如果设置了64bit不兼容功能标志，则为组描述符的大小，以字节为单位。 |
| 0x100 | __le32 | s_default_mount_opts | 默认挂载选项。更多信息请参见super_mountopts表。 |
| 0x104 | __le32 | s_first_meta_bg | 如果启用了meta_bg功能，则为第一个元数据块组。 |
| 0x108 | __le32 | s_mkfs_time | 文件系统创建时间，从纪元开始的秒数。 |
| 0x10C | __le32 | s_jnl_blocks[17] | 日志inode的i_block[]数组的备份副本在前15个元素中，i_size_high和i_size分别在第16和第17个元素中。 |
|  |  |  |仅当设置了`EXT4_FEATURE_COMPAT_64BIT`时，64位支持才有效。|
| `0x150` | `__le32` | `s_blocks_count_hi` | 块计数的高32位。 |
| `0x154` | `__le32` | `s_r_blocks_count_hi` | 保留块计数的高32位。 |
| `0x158` | `__le32` | `s_free_blocks_count_hi` | 空闲块计数的高32位。 |
| `0x15C` | `__le16` | `s_min_extra_isize` | 所有inode至少有#字节。 |
| `0x15E` | `__le16` | `s_want_extra_isize` | 新inode应保留#字节。 |
| `0x160` | `__le32` | `s_flags` | 杂项标志。更多信息请参见super_flags表。 |
| `0x164` | `__le16` | `s_raid_stride` | RAID步幅。这是在移动到下一个磁盘之前从磁盘读取或写入的逻辑块数。这会影响文件系统元数据的放置，希望使RAID存储更快。 |
| `0x166` | `__le16` | `s_mmp_interval` | 在多挂载防护(MMP)检查中等待的秒数。理论上，MMP是一种机制，用于在超级块中记录哪个主机和设备已挂载文件系统，以防止多次挂载。这个功能似乎没有实现... |
| `0x168` | `__le64` | `s_mmp_block` | 多挂载保护数据的块#。 |
| `0x170` | `__le32` | `s_raid_stripe_width` | RAID条带宽度。这是在返回到当前磁盘之前从磁盘读取或写入的逻辑块数。块分配器使用它来尝试减少RAID5/6中的读-修改-写操作数量。 |
| `0x174` | `__u8` | `s_log_groups_per_flex` | 灵活块组的大小为`2^s_log_groups_per_flex`。 |
| `0x175` | `__u8` | `s_checksum_type` | 元数据校验和算法类型。唯一有效值为1 (crc32c)。 |
| `0x176` | `__le16`| `s_reserved_pad` |  |
| `0x178` | `__le64`| `s_kbytes_written` | 在其生命周期内写入此文件系统的KiB数。 |
| `0x180` | `__le32`| `s_snapshot_inum` | 活动快照的inode号。（在`e2fsprogs/Linux`中未使用。） |
| `0x184` | `__le32`| `s_snapshot_id` | 活动快照的顺序ID。（在`e2fsprogs/Linux`中未使用。） |
| `0x188` | `__le64`| `s_snapshot_r_blocks_count` | 为活动快照的未来使用保留的块数。（在`e2fsprogs/Linux`中未使用。） |
| `0x190` | `__le32`| `s_snapshot_list` | 磁盘上快照列表头的inode号。（在`e2fsprogs/Linux`中未使用。） |
| `0x194` | `__le32`| `s_error_count` | 看到的错误数。 |
| `0x198` | `__le32`| `s_first_error_time` | 第一次发生错误的时间，从纪元开始的秒数。 |
| `0x19C` | `__le32`| `s_first_error_ino` | 第一次错误涉及的inode。 |
| `0x1A0` | `__le64`| `s_first_error_block` | 第一次错误涉及的块号。 |
| `0x1A8` | `__u8` |`s_first_error_func[32]` | 发生错误的函数名称。 |
| `0x1C8` | `__le32`| `s_first_error_line` | 发生错误的行号。 |
| `0x1CC` | `__le32`| `s_last_error_time` | 最近一次错误的时间，从纪元开始的秒数。 |
| `0x1D0` | `__le32`| `s_last_error_ino` | 最近一次错误涉及的inode。 |
| `0x1D4` | `__le32`| `s_last_error_line` | 最近一次错误发生的行号。 |
| `0x1D8` | `__le64`| `s_last_error_block` | 最近一次错误涉及的块号。 |
| `0x1E0` | `__u8` |`s_last_error_func[32]` | 最近一次错误发生的函数名称。 |
| `0x200` | `__u8` |`s_mount_opts[64]` | 挂载选项的ASCIIZ字符串。 |
| `0x240` | `__le32`| `s_usr_quota_inum` | 用户配额文件的Inode号。 |
| `0x244` | `__le32`| `s_grp_quota_inum` | 组配额文件的Inode号。 |
| `0x248` | `__le32`| `s_overhead_blocks` | 文件系统中的开销块/集群。（这个字段总是零，这意味着内核动态计算它。） |
| `0x24C` | `__le32`| `s_backup_bgs[2]` | 包含超级块备份的块组（如果`sparse_super2`） |
| `0x254` | `__u8` | `s_encrypt_algos[4]` | 使用中的加密算法。任何时候最多可以使用四种算法；有效的算法代码在下面的`super_encrypt`表中给出。 |
| `0x258` | `__u8` | `s_encrypt_pw_salt[16]` | 用于加密的`string2key`算法的盐。 |
| `0x268` | `__le32` | `s_lpf_ino` | `lost+found`的inode号 |
| `0x26C` | `__le32` | `s_prj_quota_inum` | 跟踪项目配额的inode。 |
| `0x270` | `__le32` | `s_checksum_seed` | 用于`metadata_csum`计算的校验和种子。此值为`crc32c(~0, $orig_fs_uuid)`。 |
| `0x274` | `__u8` | `s_wtime_hi` | `s_wtime`字段的高8位。 |
| `0x275` | `__u8` | `s_mtime_hi` | `s_mtime`字段的高8位。 |
| `0x276` | `__u8` | `s_mkfs_time_hi` | `s_mkfs_time`字段的高8位。 |
| `0x277` | `__u8` | `s_lastcheck_hi` | `s_lastcheck`字段的高8位。 |
| `0x278` | `__u8` | `s_first_error_time_hi` | `s_first_error_time`字段的高8位。 |
| `0x279` | `__u8` | `s_last_error_time_hi` | `s_last_error_time`字段的高8位。 |
| `0x27A` | `__u8` | `s_pad[2]` | 零填充。 |
| `0x27C` | `__le16` | `s_encoding` | 文件名字符集编码。 |
| `0x27E` | `__le16` | `s_encoding_flags` | 文件名字符集编码标志。 |
| `0x280` | `__le32` | `s_orphan_file_inum` | 孤立文件inode号。 |
| `0x284` | `__le32` | `s_reserved[94]` | 填充到块的末尾。 |
| `0x3FC` | `__le32` | `s_checksum` | 超级块校验和。 |

`superblock_state`是以下值的组合：

|   值  |  描述  |
| ------ | ----- |
|`0x0001`|干净卸载|
|`0x0002`|检测到错误|
|`0x0004`|正在恢复孤立文件|

`superblock_errors`策略是以下值之一：

| 值 | 描述 |
|---|------|
| 1 | 继续 |
| 2 | 以只读方式重新挂载 |
| 3 | 系统崩溃 |

`filesystem_creator`是以下值之一：

| 值 | 描述 |
| -- | --- | 
| 0 | Linux |
| 1 | Hurd |
| 2 | Masix |
| 3 | FreeBSD |
| 4 | Lites|

`superblock_revision`是以下值之一：

| 值 | 描述 | 
| -- | --- |
| 0 | 原始格式 | 
| 1 | v2格式，具有动态inode大小|

注意，`EXT4_DYNAMIC_REV` 指的是修订版本1或更新的文件系统。

`super_compat`字段是以下任何值的组合：

| 值 | 描述 |
| -- | --- |
| 0x1 | 目录预分配 (COMPAT_DIR_PREALLOC) |
| 0x2 | "imagic inodes"。从代码中看不清楚这是做什么的 (COMPAT_IMAGIC_INODES) | 
| 0x4 | 有日志 (COMPAT_HAS_JOURNAL) |
| 0x8 | 支持扩展属性 (COMPAT_EXT_ATTR) |
| 0x10 | 为文件系统扩展保留GDT块 (COMPAT_RESIZE_INODE)。需要RO_COMPAT_SPARSE_SUPER |
| 0x20 | 有目录索引 (COMPAT_DIR_INDEX)0x40"懒惰BG"。不在Linux内核中，似乎是用于未初始化的块组？(COMPAT_LAZY_BG) |
| 0x80 | "排除inode"。未使用 (COMPAT_EXCLUDE_INODE) |
| 0x100 | "排除位图"。似乎用于指示存在与快照相关的排除位图？在内核或`e2fsprogs`中未定义 (COMPAT_EXCLUDE_BITMAP) |
| 0x200 | 稀疏超级块，v2。如果设置了此标志，SB字段`s_backup_bgs`指向包含备份超级块的两个块组 (COMPAT_SPARSE_SUPER2) |
| 0x400 | 支持快速提交。尽管快速提交块向后不兼容，但快速提交块并不总是出现在日志中。如果日志中存在快速提交块，则设置JBD2不兼容特性(JBD2_FEATURE_INCOMPAT_FAST_COMMIT)(COMPAT_FAST_COMMIT)。|
| 0x1000 | 已分配孤儿文件。这是一种特殊文件，用于更有效地跟踪已取消链接但仍然打开的inode。当文件中可能有任何条目时，我们还会设置适当的rocompat特性(RO_COMPAT_ORPHAN_PRESENT)。 |

`super_incompat`特性字段是以下任意组合：

| 值 | 描述 |
| --- | ---- | 
| 0x1 | 压缩(INCOMPAT_COMPRESSION)。|
| 0x2 | 目录条目记录文件类型。见下面的ext4_dir_entry_2(INCOMPAT_FILETYPE)。|
| 0x4 | 文件系统需要恢复(INCOMPAT_RECOVER)。| 
| 0x8 | 文件系统有单独的日志设备(INCOMPAT_JOURNAL_DEV)。|
| 0x10 | 元块组。请参阅前面对此特性的讨论(INCOMPAT_META_BG)。| 
| 0x40 | 此文件系统中的文件使用区段(INCOMPAT_EXTENTS)。|
| 0x80 | 启用2^64块的文件系统大小(INCOMPAT_64BIT)。| 
| 0x100 | 多重挂载保护(INCOMPAT_MMP)。| 
| 0x200 | 灵活块组。请参阅前面对此特性的讨论(INCOMPAT_FLEX_BG)。|
| 0x400 | inode可用于存储大型扩展属性值(INCOMPAT_EA_INODE)。|
| 0x1000 | 目录条目中的数据(INCOMPAT_DIRDATA)。（未实现？）|
| 0x2000 | 元数据校验和种子存储在超级块中。此特性使管理员能够在文件系统挂载时更改metadata_csum文件系统的UUID；没有它，校验和定义要求重写所有元数据块(INCOMPAT_CSUM_SEED)。|
| 0x4000 | 大型目录>2GB或3级htree(INCOMPAT_LARGEDIR)。在此特性之前，目录不能大于4GiB，htree深度不能超过2级。如果启用此特性，目录可以大于4GiB，最大htree深度为3。|
| 0x8000 | inode中的数据(INCOMPAT_INLINE_DATA)。|
| 0x10000 | 文件系统上存在加密inode。(INCOMPAT_ENCRYPT)。|

`super_rocompat`特性字段是以下任意组合：

| 值 | 描述 |
| --- | ---- | 
| 0x1 | 稀疏超级块。请参阅前面对此特性的讨论(RO_COMPAT_SPARSE_SUPER)。|
| 0x2 | 此文件系统已用于存储大于2GiB的文件(RO_COMPAT_LARGE_FILE)。|
| 0x4 | 内核或e2fsprogs中未使用(RO_COMPAT_BTREE_DIR)。|
| 0x8 | 此文件系统具有以逻辑块为单位（而非512字节扇区）表示大小的文件。这意味着一个非常大的文件！(RO_COMPAT_HUGE_FILE)
| 0x10 | 组描述符有校验和。除了检测损坏外，这对于使用未初始化组的懒格式化也很有用(RO_COMPAT_GDT_CSUM)。|
| 0x20 | 表示旧的ext3 32,000子目录限制不再适用(RO_COMPAT_DIR_NLINK)。如果目录的i_links_count超过64,999，将设置为1。|
| 0x40 | 表示此文件系统上存在大型inode(RO_COMPAT_EXTRA_ISIZE)。|
| 0x80 | 此文件系统有快照(RO_COMPAT_HAS_SNAPSHOT)。|
| 0x100 | 配额(RO_COMPAT_QUOTA)。|
| 0x200 | 此文件系统支持"bigalloc"，这意味着文件区段以块的集群为单位跟踪，而不是以块为单位(RO_COMPAT_BIGALLOC)。|
| 0x400 | 此文件系统支持元数据校验和。(RO_COMPAT_METADATA_CSUM；暗示RO_COMPAT_GDT_CSUM，尽管不能设置GDT_CSUM) |
| 0x800 | 文件系统支持副本。此功能既不在内核中也不在e2fsprogs中。(RO_COMPAT_REPLICA)0x1000只读文件系统镜像；内核不会以读写方式挂载此镜像，大多数工具将拒绝写入该镜像。(RO_COMPAT_READONLY) |
| 0x2000 | 文件系统跟踪项目配额。(RO_COMPAT_PROJECT) |
| 0x8000 | 文件系统上可能存在完整性校验inode。(RO_COMPAT_VERITY) |
| 0x10000 | 表示孤儿文件可能有有效的孤儿条目，因此我们需要在挂载文件系统时清理它们(RO_COMPAT_ORPHAN_PRESENT)。|

`s_def_hash_version`字段是以下之一：

| 值 | 描述 |
| -- | --- |
| 0x0 | 传统。|
| 0x1 | 半MD4。| 
| 0x2 | Tea。|
| 0x3 | 传统，无符号。|
| 0x4 | 半MD4，无符号。| 
| 0x5 | Tea，无符号。|

`s_default_mount_opts`字段是以下任意组合：

| 值 | 描述 |
| -- | --- |
| 0x0001 | 在（重新）挂载时打印调试信息。(EXT4_DEFM_DEBUG) |
| 0x0002 | 新文件采用包含目录的gid（而不是当前进程的fsgid）。(EXT4_DEFM_BSDGROUPS) |
| 0x0004 | 支持用户空间提供的扩展属性。(EXT4_DEFM_XATTR_USER) |
| 0x0008 | 支持POSIX访问控制列表(ACL)。(EXT4_DEFM_ACL) |
| 0x0010 | 不支持32位UID。(EXT4_DEFM_UID16)|
| 0x0020 | 所有数据和元数据都提交到日志。(EXT4_DEFM_JMODE_DATA) |
| 0x0040 | 在元数据提交到日志之前，所有数据都刷新到磁盘。(EXT4_DEFM_JMODE_ORDERED)|
| 0x0060 | 不保留数据顺序；数据可能在元数据写入后才写入。(EXT4_DEFM_JMODE_WBACK) | 
| 0x0100 | 禁用写入刷新。(EXT4_DEFM_NOBARRIER) |
| 0x0200 | 跟踪文件系统中哪些块是元数据，因此不应用作数据块。此选项将在3.18上默认启用，希望如此。(EXT4_DEFM_BLOCK_VALIDITY) |
| 0x0400 | 启用DISCARD支持，告知存储设备哪些块变得未使用。(EXT4_DEFM_DISCARD)|
| 0x0800 | 禁用延迟分配。(EXT4_DEFM_NODELALLOC)|

`s_flags`字段是以下任意组合：

| 值 | 描述 |
| -- | --- |
| 0x0001 | 使用带符号的目录哈希。|
| 0x0002 | 使用无符号的目录哈希。|
| 0x0004 | 用于测试开发代码。|

`s_encrypt_algos`列表可以包含以下任意一项：

| 值 | 描述 |
| -- | --- |
| 0 | 无效算法(ENCRYPTION_MODE_INVALID)。|
| 1 | XTS模式下的256位AES(ENCRYPTION_MODE_AES_256_XTS)。|
| 2 | GCM模式下的256位AES(ENCRYPTION_MODE_AES_256_GCM)。|
| 3 | CBC模式下的256位AES(ENCRYPTION_MODE_AES_256_CBC)。|

超级块的总大小为1024字节。

## 3.2. 块组描述符

文件系统上的每个块组都有一个与之关联的描述符。如上面布局部分所述，组描述符（如果存在）是块组中的第二个项目。标准配置是每个块组包含完整的块组描述符表副本，除非设置了`sparse_super`特性标志。

请注意组描述符如何记录位图和inode表的位置（即它们可以浮动）。这意味着在块组内，唯一具有固定位置的数据结构是超级块和组描述符表。`flex_bg`机制利用此属性将多个块组分组到一个flex组中，并在flex组的第一个组中将所有组的位图和inode表布置成一个长运行。

如果设置了`meta_bg`特性标志，则多个块组将被分组到一个元组中。但是，请注意，在`meta_bg`情况下，较大元组内的第一个和最后两个块组仅包含元组内组的组描述符。

`flex_bg`和`meta_bg`特性似乎并不互斥。

在ext2、ext3和ext4（未启用64bit特性时），块组描述符仅为32字节长，因此在`bg_checksum`处结束。在启用了64bit特性的ext4文件系统上，块组描述符至少扩展到下面描述的64字节；大小存储在超级块中。

如果设置了`gdt_csum`但未设置`metadata_csum`，则块组校验和是FS UUID、组号和组描述符结构的crc16。如果设置了`metadata_csum`，则块组校验和是FS UUID、组号和组描述符结构的校验和的低16位。块和inode位图校验和是针对FS UUID、组号和整个位图计算的。

块组描述符按照`struct ext4_group_desc`布局。

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ---- |
| 0x0 | __le32 | bg_block_bitmap_lo | 块位图位置的低32位。 |
| 0x4 | __le32 | bg_inode_bitmap_lo | inode位图位置的低32位。|
| 0x8 | __le32 | bg_inode_table_lo | inode表位置的低32位。|
| 0xC | __le16| bg_free_blocks_count_lo | 空闲块计数的低16位。 |
| 0xE | __le16 | bg_free_inodes_count_lo | 空闲inode计数的低16位。|
| 0x10 | __le16 | bg_used_dirs_count_lo | 目录计数的低16位。 |
| 0x12 | __le16 | bg_flags | 块组标志。参见下面的bgflags表。 |
| 0x14 | __le32 | bg_exclude_bitmap_lo | 快照排除位图位置的低32位。|
| 0x18 | __le16 | bg_block_bitmap_csum_lo | 块位图校验和的低16位。|
| 0x1A | __le16 | bg_inode_bitmap_csum_lo | inode位图校验和的低16位。|
| 0x1C | __le16 | bg_itable_unused_lo | 未使用inode计数的低16位。如果设置，我们无需扫描超过此组inode表中的(sb.s_inodes_per_group - gdt.bg_itable_unused)个条目。
| 0x1E | __le16 | bg_checksum | 组描述符校验和；如果设置了RO_COMPAT_GDT_CSUM特性，则为crc16(sb_uuid+group_num+bg_desc)，或者如果设置了RO_COMPAT_METADATA_CSUM特性，则为crc32c(sb_uuid+group_num+bg_desc) & 0xFFFF。在计算crc16校验和时跳过bg_desc中的bg_checksum字段，如果使用crc32c校验和则设置为零。|
| |  | | 以下字段仅在启用64bit特性且s_desc_size > 32时存在。|
| 0x20 | __le32 | bg_block_bitmap_hi | 块位图位置的高32位。|
| 0x24 | __le32 | bg_inode_bitmap_hi | inode位图位置的高32位。|
| 0x28 | __le32 | bg_inode_table_hi  |inode表位置的高32位。| 
| 0x2C | __le16 | bg_free_blocks_count_hi | 空闲块计数的高16位。|
| 0x2E | __le16 | bg_free_inodes_count_hi | 空闲inode计数的高16位。|
| 0x30 | __le16 | bg_used_dirs_count_hi | 目录计数的高16位。|
| 0x32 | __le16 | bg_itable_unused_hi | 未使用inode计数的高16位。|
| 0x34 | __le32 | bg_exclude_bitmap_hi | 快照排除位图位置的高32位。|
| 0x38 | __le16 | bg_block_bitmap_csum_hi |块位图校验和的高16位。|
| 0x3A | __le16 | bg_inode_bitmap_csum_hi | inode位图校验和的高16位。|
| 0x3C | __u32 | bg_reserved | 填充至64字节。|

块组标志可以是以下任意组合：

| 值 | 描述 |
| -- | --- |
| 0x1 | inode表和位图未初始化(EXT4_BG_INODE_UNINIT)。|
| 0x2 | 块位图未初始化(EXT4_BG_BLOCK_UNINIT)。|
| 0x4 | inode表已置零(EXT4_BG_INODE_ZEROED)。|

## 3.3. 块和inode位图

数据块位图跟踪块组内数据块的使用情况。

inode位图记录inode表中哪些条目正在使用。

与大多数位图一样，一个位表示一个数据块或inode表条目的使用状态。这意味着块组大小为8 * 逻辑块中的字节数。

注意：如果为给定块组设置了`BLOCK_UNINIT`，内核和e2fsprogs代码的各个部分会假装块位图包含零（即组中的所有块都是空闲的）。然而，不一定意味着没有块在使用——如果设置了`meta_bg`，位图和组描述符位于组内。不幸的是，`ext2fs_test_block_bitmap2()`将对这些位置返回'0'，这会产生令人困惑的debugfs输出。

> BLOCK_UNINIT标志作用的时机是在文件系统的操作过程中，特别是以下几个场景:
> 
> 文件系统挂载时 - 当内核挂载文件系统并读取块组描述符时，如果发现某个块组设置了BLOCK_UNINIT标志，内核会假设该块组的所有块都是空闲的，而不会实际读取块位图的内容。
>
> 文件系统工具运行时 - 当e2fsprogs工具包中的程序(如e2fsck、debugfs等)处理文件系统时，它们会检查这个标志并相应地调整行为。
>
> 分配新块时 - 当文件系统需要分配新块时，如果检测到一个块组有BLOCK_UNINIT标志，它会首先初始化该块组的块位图，然后移除这个标志，表示该块组现在已被初始化。

## 3.4. Inode表

Inode表在mkfs时静态分配。每个块组描述符指向表的开始，超级块记录每组的inode数量。有关更多信息，请参阅inode部分。

## 3.5. Multiple Mount Protection

多重挂载保护(MMP)是一种功能，用于防止多个主机尝试同时使用文件系统。当打开文件系统（用于挂载或fsck等）时，节点（称为节点A）上运行的MMP代码检查序列号。如果序列号是`EXT4_MMP_SEQ_CLEAN`，则继续打开。如果序列号是`EXT4_MMP_SEQ_FSCK`，则说明fsck（希望）正在运行，打开立即失败。否则，打开代码将等待指定MMP检查间隔的两倍时间，然后再次检查序列号。如果序列号已更改，则文件系统在另一台机器上处于活动状态，打开失败。如果MMP代码通过了所有这些检查，将生成一个新的MMP序列号并写入MMP块，然后继续挂载。

当文件系统处于活动状态时，内核设置一个定时器，以指定的MMP检查间隔重新检查MMP块。要执行重新检查，重新读取MMP序列号；如果它与内存中的MMP序列号不匹配，则另一个节点（节点B）已挂载文件系统，节点A将文件系统重新挂载为只读。如果序列号匹配，则序列号在内存和磁盘上都递增，重新检查完成。

每当打开操作成功时，主机名和设备文件名都会写入MMP块。MMP代码不使用这些值；它们纯粹是为了提供信息。

校验和是针对FS UUID和MMP结构计算的。MMP结构(struct mmp_struct)如下：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ---- |
| 0x0 | __le32 | mmp_magic | MMP的魔术数字，0x004D4D50（"MMP"）。
| 0x4 | __le32 | mmp_seq | 序列号，定期更新。|
| 0x8 | __le64 | mmp_time | MMP块最后更新的时间。|
| 0x10 | char[64] | mmp_nodename | 打开文件系统的节点的主机名。|
| 0x50 | char[32] | mmp_bdevname | 文件系统的块设备名称。 |
| 0x70 | __le16 | mmp_check_interval | MMP重新检查间隔，以秒为单位。|
| 0x72 | __le16 | mmp_pad1 | 零。|
| 0x74 | __le32[226] | mmp_pad2 | 零。|
| 0x3FC | __le32 | mmp_checksum | MMP块的校验和。|

## 3.6. Journal (jbd2)

在ext3中引入，`ext4`文件系统使用日志来在系统崩溃的情况下保护文件系统免受元数据不一致的影响。最多可以在文件系统内保留10,240,000个文件系统块（有关日志大小限制的更多详细信息，请参见`man mke2fs(8)`），作为尽快将"重要"数据写入磁盘的地方。一旦重要数据事务完全写入磁盘并从磁盘写入缓存中刷新，数据提交的记录也会写入日志。在稍后的某个时间点，日志代码将事务写入其在磁盘上的最终位置（这可能涉及大量寻道或大量小型读写擦除）然后才擦除提交记录。如果系统在第二次缓慢写入期间崩溃，可以重放日志直到最新的提交记录，保证无论通过日志写入磁盘的内容的原子性。这样做的效果是保证文件系统不会在元数据更新过程中卡住。

出于性能原因，ext4默认只通过日志写入文件系统元数据。这意味着在崩溃后，文件数据块不能保证处于任何一致的状态。如果这种默认保证级别（`data=ordered`）不令人满意，有一个挂载选项可以控制日志行为。如果`data=journal`，所有数据和元数据都通过日志写入磁盘。这更慢但最安全。如果`data=writeback`，在元数据通过日志写入磁盘之前，不会将脏数据块刷新到磁盘。

在`data=ordered`模式下，Ext4还支持快速提交，这有助于显著减少提交延迟。默认的`data=ordered`模式通过将元数据块记录到日志来工作。在快速提交模式下，Ext4只存储需要重新创建受影响元数据的最小增量，存储在与JBD2共享的快速提交空间中。一旦快速提交区域填满或者无法进行快速提交或者JBD2提交计时器超时，Ext4执行传统的完整提交。完整提交使其之前发生的所有快速提交无效，从而使快速提交区域为进一步的快速提交腾出空间。此功能需要在mkfs时启用。

日志inode通常是inode 8。日志inode的前68个字节在ext4超级块中复制。日志本身是文件系统内的正常（但隐藏）文件。该文件通常消耗整个块组，尽管mke2fs尝试将其放在磁盘中间。

jbd2中的所有字段都以大端序写入磁盘。这与ext4相反。

注意：ext4和ocfs2都使用jbd2。

嵌入在ext4文件系统中的日志的最大大小是2^32块。jbd2本身似乎并不关心。

### 3.6.1. 布局

一般来说，日志具有以下格式：

| 超级块 | 描述符块 (数据块或撤销块) \[更多数据或撤销 \] 提交块 | \[更多事务...\] |
| ----- | ----- | ----- |
| | 一个事务 | |

请注意，事务以描述符和一些数据开始，或者以块撤销列表开始。完成的事务总是以提交结束。如果没有提交记录（或校验和不匹配），事务将在重放期间被丢弃。

### 3.6.2. 外部日志

可选地，可以使用外部日志设备创建ext4文件系统（与使用保留inode的内部日志相对）。在这种情况下，在文件系统设备上，`s_journal_inum`应该为零，`s_journal_uuid`应该设置。在日志设备上，ext4超级块将位于通常的位置，具有匹配的UUID。日志超级块将位于超级块之后的下一个完整块中。

| 1024字节的填充 | ext4超级块 | 日志超级块 | 描述符块 (数据块或撤销块) \[更多数据或撤销\] 提交块 | \[更多事务...\] | 
| --- | --- | --- | --- | --- |
| | | | 一个事务 | |

### 3.6.3. 块头

日志中的每个块都以一个共同的12字节头`struct journal_header_s`开始：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ---- |
| 0x0 | __be32 | h_magic | jbd2魔术数字，0xC03B3998。 |
| 0x4 | __be32 | h_blocktype | 描述此块包含的内容。参见下面的`jbd2_blocktype`表。|
| 0x8 | __be32 | h_sequence | 与此块关联的事务ID。|

日志块类型可以是以下任意一种：

| 值 | 描述 |
| --- | ---- |
| 1 | 描述符。此块位于一系列数据块之前，这些数据块在事务期间通过日志写入。|
| 2 | 块提交记录。此块表示事务的完成。|
| 3 | 日志超级块，v1。4日志超级块，v2。|
| 5 | 块撤销记录。这通过使日志能够跳过写入随后被重写的块来加速恢复。|

### 3.6.4. 超级块

与ext4相比，日志的超级块要简单得多。其中保存的关键数据是日志的大小，以及在哪里可以找到事务日志的开始。

日志超级块记录为`struct journal_superblock_s`，长度为1024字节：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ---- |
|  |  |  | 描述日志的静态信息。|
| 0x0 | journal_header_t (12 bytes) | s_header | 标识这是超级块的通用头。|
| 0xC | __be32 | s_blocksize | 日志设备块大小。| 0x10 | __be32 | s_maxlen | 此日志中的块总数。|
| 0x14 | __be32 | s_first | 日志信息的第一个块。描述日志当前状态的动态信息。|
| 0x18 | __be32 | s_sequence | 日志中预期的第一个提交ID。|
| 0x1C | __be32 | s_start | 日志开始的块号。与注释相反，此字段为零并不意味着日志是干净的！|
| 0x20 | __be32 | s_errno | 错误值，由jbd2_journal_abort()设置。以下字段仅在v2超级块中有效。|
| 0x24 | __be32 | s_feature_compat | 兼容特性集。参见下面的jbd2_compat表。| 
| 0x28 | __be32 | s_feature_incompat | 不兼容特性集。参见下面的jbd2_incompat表。|
| 0x2C | __be32 | s_feature_ro_compat | 只读兼容特性集。目前没有任何这样的特性。|
| 0x30 | __u8 | s_uuid[16] | 日志的128位uuid。在挂载时与ext4超级块中的副本进行比较。|
| 0x40 | __be32 | s_nr_users | 共享此日志的文件系统数量。|
| 0x44 | __be32 | s_dynsuper | 动态超级块副本的位置。（不使用？）|
| 0x48 | __be32 | s_max_transaction | 每个事务的日志块限制。（不使用？）|
| 0x4C | __be32 | s_max_trans_data | 每个事务的数据块限制。（不使用？）|
| 0x50 | __u8s | _checksum_type  |用于日志的校验和算法。有关更多信息，请参见jbd2_checksum_type。|
| 0x51 | __u8[3] | s_padding2 | 
| 0x54 | __be32 | s_num_fc_blocks | 日志中的快速提交块数量。|
| 0x58 | __be32 | s_head | 日志头（第一个未使用的块）的块号，仅在日志为空时更新。|
| 0x5C | __u32 | s_padding[40] | | 
| 0xFC | __be32 | s_checksum |整个超级块的校验和，此字段设置为零。|
| 0x100 | __u8 | s_users[16*48] | 共享日志的所有文件系统的ID。e2fsprogs/Linux不允许共享外部日志，但我想Lustre（或ocfs2？）使用jbd2代码可能会。|

日志兼容特性是以下任意组合：

| 值 | 描述 |
| --- | ---- |
| 0x1 | 日志维护数据块的校验和。(JBD2_FEATURE_COMPAT_CHECKSUM)|

日志不兼容特性是以下任意组合：

| 值 | 描述 |
| --- | --- |
| 0x1 | 日志有块撤销记录。(JBD2_FEATURE_INCOMPAT_REVOKE)|
| 0x2 | 日志可以处理64位块号。(JBD2_FEATURE_INCOMPAT_64BIT)|
| 0x4 | 日志异步提交。(JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT) |
| 0x8 | 该日志使用校验和磁盘格式的v2版本。每个日志元数据块获得自己的校验和，描述符表中的块标签包含日志中每个数据块的校验和。(JBD2_FEATURE_INCOMPAT_CSUM_V2)|
| 0x10 | 该日志使用校验和磁盘格式的v3版本。这与v2相同，但日志块标签大小是固定的，不管块号的大小如何。(JBD2_FEATURE_INCOMPAT_CSUM_V3)|
| 0x20 | 日志有快速提交块。(JBD2_FEATURE_INCOMPAT_FAST_COMMIT)|

日志校验和类型代码是以下之一。crc32或crc32c是最可能的选择。

| 值 | 描述 |
| -- | ---- |
| 1 | CRC32 |
| 2 | MD5 | 
| 3 | SHA1 |
| 4 | CRC32C|

### 3.6.5. 描述符块

描述符块包含一组日志块标签，描述了在日志中跟随的数据块的最终位置。描述符块是开放编码的，而不是由数据结构完全描述，但这里是块结构。描述符块至少消耗36字节，但使用完整的块：

| 偏移量 | 类型 | 名称 | 描述符 |
| ----- | ---- | ---- | ----- |
| 0x0 | journal_header_t | (开放编码) | 通用块头。|
| 0xC | struct journal_block_tag_s | 开放编码数组[] | 足够的标签，要么填满块，要么描述跟随此描述符块的所有数据块。|

日志块标签有以下任何格式，取决于设置的日志特性和块标签标志。

如果设置了`JBD2_FEATURE_INCOMPAT_CSUM_V3`，则日志块标签定义为`struct journal_block_tag3_s`，其外观如下。大小为16或32字节。

| 偏移量 | 类型 | 名称 | 描述符 |
| ----- | ---- | ---- | ----- |
| 0x0 | __be32 | t_blocknr  |对应数据块应在磁盘上结束的位置的低32位。|
| 0x4 | __be32 | t_flags  |与描述符一起的标志。有关更多信息，请参见jbd2_tag_flags表。|
| 0x8 | __be32 | t_blocknr_high | 对应数据块应在磁盘上结束的位置的高32位。如果未启用JBD2_FEATURE_INCOMPAT_64BIT，则为零。|
| 0xC | __be32 | t_checksum | 日志UUID、序列号和数据块的校验和。|
|     |        |            |此字段似乎是开放编码的。它总是在t_checksum之后。如果设置了"same UUID"标志，则不存在此字段。|
| 0x8或0xC | char | uuid[16]  |与此标签一起的UUID。此字段似乎是从struct journal_s中的j_uuid字段复制的，但只有tune2fs才触及该字段。

日志标签标志是以下任意组合：

| 值 | 描述 |
| -- | --- |
| 0x1 | 磁盘上的块是转义的。数据块的前四个字节恰好与jbd2魔术数字匹配。|
| 0x2 | 该块与前一个具有相同的UUID，因此省略了UUID字段。|
| 0x4 | 数据块被事务删除。(不使用？) |
| 0x8 | 这是此描述符块中的最后一个标签。|

如果未设置`JBD2_FEATURE_INCOMPAT_CSUM_V3`，则日志块标签定义为`struct journal_block_tag_s`，其外观如下。大小为8、12、24或28字节：

| 偏移量 | 类型 | 名称 | 描述符 |
| ----- | ---- | ---- | ----- |
| 0x0 | __be32 | t_blocknr | 对应数据块应在磁盘上结束的位置的低32位。|
| 0x4 | __be16 | t_checksum | 日志UUID、序列号和数据块的校验和。注意，只存储低16位。|
| 0x6 | __be16 | t_flags | 与描述符一起的标志。参见jbd2_tag_flags表。| 
| | | |以下字段仅在超级块表示支持64位块号时存在。|
|0x8 | __be32 | t_blocknr_high | 对应数据块应在磁盘上结束的位置的高32位。|
| | | | 此字段似乎是开放编码的。它总是在t_flags或t_blocknr_high之后。如果设置了"same UUID"标志，则不存在此字段。|
| 0x8或0xC | char | uuid[16] | 与此标签一起的UUID。此字段似乎是从struct journal_s中的j_uuid字段复制的，但只有tune2fs才触及该字段。|

如果设置了JBD2_FEATURE_INCOMPAT_CSUM_V2或JBD2_FEATURE_INCOMPAT_CSUM_V3，则块的末尾是`struct jbd2_journal_block_tail`，其样子如下：

| 偏移量 | 类型 | 名称 | 描述符 |
| ----- | ---- | ---- | ----- |
| 0x0 | __be32 | t_checksum| 日志UUID+描述符块的校验和，此字段设置为零。|

### 3.6.6. 数据块

通常，通过日志写入磁盘的数据块在描述符块之后一字不差地写入日志文件。但是，如果块的前四个字节与jbd2魔术数字匹配，则这四个字节被替换为零，并在描述符块标签中设置"escaped"标志。

### 3.6.7. 撤销块

撤销块用于防止在重放块早期事务中。这用于标记曾经记录在日志中但不再从日志中重放的块。通常，如果元数据块被释放并重新分配为文件数据块，就会发生这种情况；在这种情况下，如果在文件块写入磁盘后进行日志重放，将会导致损坏。

注意：此机制不用于表达"此日志块被此其他日志块取代"，作者（djwong）错误地认为。添加到事务中的任何块都将导致该块的所有现有撤销记录被删除。

> 撤销块的主要目的是防止在重放早期事务时导致的潜在数据损坏。它们专门标记那些"曾经记录在日志中但不再应该从日志重放"的块。

撤销块在`struct jbd2_journal_revoke_header_s`中描述，至少为16字节长，但使用完整块：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ----- |
| 0x0 | journal_header_t | r_header | 通用块头。|
| 0xC | __be32| r_count | 此块中使用的字节数。|
| 0x10 | __be32或__be64 | blocks[0] | 要撤销的块。|

在`r_count`之后是一个线性数组，包含由此事务有效撤销的块号。如果超级块公布64位块号支持，则每个块号的大小为8字节，否则为4字节。

如果设置了`JBD2_FEATURE_INCOMPAT_CSUM_V2`或`JBD2_FEATURE_INCOMPAT_CSUM_V3`，则撤销块的末尾是`struct jbd2_journal_revoke_tail`，格式如下：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ----- |
| 0x0 | __be32 | r_checksum | 日志UUID+撤销块的校验和 |

### 3.6.8. 提交块

提交块是一个哨兵，表示事务已完全写入日志。一旦此提交块到达日志，存储在此事务中的数据就可以写入其在磁盘上的最终位置。

提交块由`struct commit_header`描述，长度为32字节（但使用完整块）：

|偏移量|类型|名称|描述符|
| ----- | ---- | ---- | ----- |
| 0x0 | journal_header_s | (开放编码) | 通用块头。|
| 0xC | unsigned char | h_chksum_type | 用于验证事务中数据块完整性的校验和类型。参见jbd2_checksum_type了解更多信息。|
| 0xD | unsigned char | h_chksum_size | 校验和使用的字节数。很可能是4。|
| 0xE | unsigned char | h_padding[2] | | 
| 0x10 | __be32 | h_chksum[JBD2_CHECKSUM_BYTES] | 32字节的空间用于存储校验和。如果设置了JBD2_FEATURE_INCOMPAT_CSUM_V2或JBD2_FEATURE_INCOMPAT_CSUM_V3，则第一个__be32是日志UUID和整个提交块的校验和，此字段为零。如果设置了JBD2_FEATURE_COMPAT_CHECKSUM，则第一个__be32是已写入事务的所有块的crc32。|
| 0x30 | __be64 | h_commit_sec | 事务提交的时间，以纪元以来的秒数表示。|
| 0x38 | __be32 | h_commit_nsec | 上述时间戳的纳秒分量。|

### 3.6.9. 快速提交

快速提交区域组织为标签长度值的日志。每个TLV的开头都有一个`struct ext4_fc_tl`，存储整个字段的标签和长度。它后面跟着特定于标签的可变长度值。以下是支持的标签及其含义的列表：

| 标签 | 含义 | 值结构 | 描述 |
| --- | ---- | ------ | ---- |
| EXT4_FC_TAG_HEAD | 快速提交区域头 | struct ext4_fc_head | 存储应在其后应用这些快速提交的事务的TID。|
| EXT4_FC_TAG_ADD_RANGE | 向inode添加范围 | struct ext4_fc_add_range | 存储inode号和要添加到此inode中的范围|
| EXT4_FC_TAG_DEL_RANGE | 从inode中删除逻辑偏移 | struct ext4_fc_del_range | 存储inode号和需要删除的逻辑偏移范围|
| EXT4_FC_TAG_CREAT | 为新创建的文件创建目录条目 | struct ext4_fc_dentry_info | 存储父inode号、inode号以及新创建文件的目录条目| 
| EXT4_FC_TAG_LINK | 将目录条目链接到inodestruct | ext4_fc_dentry_info | 存储父inode号、inode号和目录条目
| EXT4_FC_TAG_UNLINK | 取消链接inode的目录条目 | struct ext4_fc_dentry_info | 存储父inode号、inode号和目录条目EXT4_FC_TAG_PAD填充（未使用区域）无快速提交区域中未使用的字节。|
| EXT4_FC_TAG_TAIL | 标记快速提交的结束 | struct ext4_fc_tail | 存储提交的TID，此标签表示结束的快速提交的CRC |

### 3.6.10. 快速提交重放幂等性

快速提交标签本质上是幂等的，前提是恢复代码遵循某些规则。提交路径在提交时遵循的指导原则是，它存储特定操作的结果，而不是存储过程。

让我们考虑这个重命名操作：'mv /a /b'。假设目录条目'/a'与inode 10关联。在快速提交期间，我们不是将此操作存储为"将a重命名为b"的过程，而是将生成的文件系统状态存储为结果的"系列"：

+ 链接目录条目b到inode 10
+ 取消链接目录条目a
+ 具有有效引用计数的inode 10

现在，当恢复代码运行时，它需要在文件系统上"强制执行"此状态。这就是保证快速提交重放幂等性的原因。

让我们以一个不是幂等的过程为例，看看快速提交如何使其变为幂等。考虑以下操作序列：

1. `rm A`
2. `mv B A`
3. `read A`

如果我们按原样存储此操作序列，则重放不是幂等的。假设在重放期间，我们在(2)之后崩溃。在第二次重放期间，文件A（实际上是作为"mv B A"操作的结果创建的）将被删除。因此，当我们尝试读取A时，名为A的文件将不存在。所以，这个操作序列不是幂等的。然而，如上所述，快速提交不是存储过程，而是存储每个过程的结果。因此，上述过程的快速提交日志将如下所示：

（假设重放前目录条目A链接到inode 10，目录条目B链接到inode 11）

1. 取消链接A
2. 将A链接到inode 11
3. 取消链接B
4. Inode 11

如果我们在(3)之后崩溃，我们将有文件A链接到inode 11。在第二次重放期间，我们将删除文件A（inode 11）。但我们会将其重新创建并使其指向inode 11。我们找不到B，所以我们将跳过该步骤。此时，inode 11的引用计数不可靠，但这通过最后一个inode 11标签的重放得到修复。因此，通过将非幂等过程转换为一系列幂等结果，快速提交确保了重放期间的幂等性。

### 3.6.11. Journal Checkpoint

检查点日志确保所有事务及其关联的缓冲区都提交到磁盘。等待进行中的事务并将其包含在检查点中。检查点在文件系统关键更新期间内部使用，包括日志恢复、文件系统调整大小和释放`journal_t`结构。

> 检查点的主要功能是确保所有事务及其关联的缓冲区都已提交到磁盘。检查点会等待所有进行中的事务完成，并将它们包含在检查点处理中。

可以通过`ioctl EXT4_IOC_CHECKPOINT`从用户空间触发日志检查点。此ioctl接受单个u64参数作为标志。目前，支持三个标志。首先，`EXT4_IOC_CHECKPOINT_FLAG_DRY_RUN`可用于验证ioctl的输入。如果有任何无效输入，它返回错误，否则在不执行任何检查点的情况下返回成功。这可用于检查系统上是否存在ioctl，并验证参数或标志没有问题。其他两个标志是`EXT4_IOC_CHECKPOINT_FLAG_DISCARD`和`EXT4_IOC_CHECKPOINT_FLAG_ZEROOUT`。这些标志分别导致日志块在日志检查点完成后被丢弃或填充零。`EXT4_IOC_CHECKPOINT_FLAG_DISCARD`和`EXT4_IOC_CHECKPOINT_FLAG_ZEROOUT`不能同时设置。在对系统进行快照或遵守内容删除SLO时，此ioctl可能很有用。

## 3.7. Orphan file

在Unix中，可能有从目录层次结构中取消链接但仍然活着的inode，因为它们是打开的。在崩溃的情况下，文件系统必须清理这些inode，否则它们（以及从它们引用的块）会泄漏。同样，如果我们截断或扩展文件，我们可能无法在单个日志事务中执行操作。在这种情况下，我们将inode追踪为孤儿，以便在崩溃的情况下，分配给文件的额外块会被截断。

> 注：当启用了孤儿文件特性(COMPAT_ORPHAN_FILE)时，文件系统会使用一个特殊的inode来记录孤儿文件信息。这个特殊inode通过超级块中的`s_orphan_file_inum`字段引用，并包含几个数据块来存储孤儿文件信息。

传统上，ext4以单链表的形式跟踪孤儿inode，其中超级块包含最后一个孤儿inode的inode号（`s_last_orphan`字段），然后每个inode包含先前孤立的inode的inode号（我们为此重载`i_dtime inode`字段）。然而，这个文件系统全局单链表对于导致大量创建孤儿inode的工作负载来说是一个可扩展性瓶颈。启用孤儿文件特性(`COMPAT_ORPHAN_FILE`)时，文件系统有一个特殊的inode（通过超级块中的`s_orphan_file_inum`引用），带有几个块。这些块中的每一个都有一个结构：

| 偏移量 | 类型 | 名称 | 描述 |
| ----- | ---- | ---- | ---- |
| 0x0 | __le32条目数组 | 孤儿inode条目 | 每个__le32条目要么为空（0），要么包含孤儿inode的inode号。|
| blocksize-8 | __le32 | ob_magic | 存储在孤儿块尾部的魔术值(0x0b10ca04)|
| blocksize-4 | __le32 | ob_checksum | 孤儿块的校验和。|

当具有孤儿文件特性的文件系统可写挂载时，我们在超级块中设置`RO_COMPAT_ORPHAN_PRESENT`特性，以指示可能有有效的孤儿条目。如果在挂载文件系统时看到此特性，我们读取整个孤儿文件并像往常一样处理所有找到的孤儿inode。当干净地卸载文件系统时，我们删除`RO_COMPAT_ORPHAN_PRESENT`特性，以避免不必要地扫描孤儿文件，并使文件系统与旧内核完全兼容。