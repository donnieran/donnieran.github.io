---
layout:     post
title:      "linux虚拟文件系统"
subtitle:   "Everything is a file"
date:       2016-04-12 20:19:00
author:     "Don"
header-img: "img/black.jpg"
tags:
    - OS
    - linux
    - VFS	
---

## 引言

虚拟文件系统，即VFS(Virtual File system),是linux内核中的一个软件抽象层。它通过一些通用的数据结构和方法实现对不同文件系统提供统一接口机制，从而实现“一切皆文件”的Unix/Linux哲学。
Linux中允许不同的文件系统共存，通过VFS提供的一套统一I/O系统调用即可对Linux系统中任何的文件进行操作而无需考虑具体的文件格式，同时支持跨平台的文件操作如拷贝等。

**VFS的作用如下:**

* 向上，对应用层提供一个标准的文件操作接口；
* 对下，对文件系统提供一个标准的接口，以便其他操作系统的文件系统可以方便的移植到Linux上；
* VFS内部则通过一系列高效的管理机制，比如inode cache, dentry cache 以及文件系统的预读等技术，使得底层文件系统不需沉溺到复杂的内核操作，即可获得高性能；
* 此外VFS把一些复杂的操作尽量抽象到VFS内部，使得底层文件系统实现更简单。

由于源码太多，不能具细，此文将通过介绍几个重要数据结构如inode、dentry等来解析VFS的抽象机制，然后分析一个sys_open()系统调用来明晰由应用层到内核底层的调用关系和逻辑分层。

---

## VFS对象及数据结构

VFS的构建是采用面对对象的思想的，一系列的数据结构指代了普遍的文件模型，这些数据结构就类似于对象。所以VFS是用C语言实现面对对象编程的一个很好范例。这些数据结构包含了数据及文件系统实现的操作这些数据的函数集合。VFS的几个基础的对象类型为：

* superblock:超级块对象，表示一个指定的安装了的文件系统
* inode:文件节点对象，表示一个具体的文件
* dentry: 目录项对象， 表示一个目录项，即为路径中以"/"分割的的单一组成部分。
* file: 文件对象，表示一个已经打开的与某进程相关联的文件。

特别说明，dentry指代的不仅仅是目录项，还指代具体的文件，VFS把目录项也是一种类型的普通文件。
在上述的基础数据结构中都包含操作函数集对象，这些对象描述内核操作这些基础数据结构的方法。

* super_operation: 包含执行低层次的对于文件系统及他们的文件节点的操作，如write_inode(),sync_fs()等。
* inode_operation: 包含了操作特定文件的方法，如create(),link()。
* dentry_operation: 包含了操作特定目录项的方法，如d_compare(),d_delete。
* file_operation: 包含了操作某一打开了的文件的方法，如read(),write()。

先看一下典型的linux文件系统EXT2在磁盘中文件的组成形式：

**图1：ext2磁盘文件系统**

![](/img/vfs/fsondisk.jpg)

#### 超级块(linux/fs.h)

**超级块对象：**超级块对象是由各自文件系统实现的用来存储描述具体文件系统的信息，此内核对象在文件系统安装时内核根据读入磁盘上的相关信息填充各个字段，而某些不依赖与磁盘的文件系统如sysfs是由内核在运行中生成此对象,实现创建、管理、摧毁超级块对象的函数位于fs/super.c文件。这个对象通常相当于具体磁盘文件系统中的的超级块或文件系统控制块，超级块或文件系统控制块通常存在于磁盘上的某一特定扇区。以上图1ext2文件系统为例，某磁盘分区的第一个块为自举块（引导块），接下来就是超级块。
{% highlight rouge %}
struct super_block {
	struct list_head	s_list;		/* 链接所有superblocks */
	dev_t			s_dev;		/* 搜索索引（index, 标识符） */
	unsigned char		s_blocksize_bits; /*以位为单位的块的大小 */
	unsigned long		s_blocksize; /* 以字节为单位的块的大小 */
	loff_t			s_maxbytes;	/* 文件的最大字节数 */
	struct file_system_type	*s_type; /* 文件系统类型 */
	const struct super_operations	*s_op; /* 操作函数集 */
	const struct dquot_operations	*dq_op; /* 磁盘限额方法 */
	const struct quotactl_ops	*s_qcop; /* 限额控制方法 */
	const struct export_operations *s_export_op; /* 导出方法 */
	unsigned long		s_flags; /* 挂在标志 */
	unsigned long		s_iflags;	/* internal SB_I_* flags */
	unsigned long		s_magic; /* 文件系统的幻数 */
	struct dentry		*s_root; /* 挂在目录 */
	struct rw_semaphore	s_umount; /* 卸载信号量*/
	int			s_count; /* 超级块引用计数 */
	atomic_t		s_active; /* 活动的引用计数 */
#ifdef CONFIG_SECURITY
	void                    *s_security; /* 安全模块 */
#endif
	const struct xattr_handler **s_xattr; /* 扩展的属性操作 */

	struct hlist_bl_head	s_anon;		/* anonymous dentries for (nfs) exporting */
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
	struct block_device	*s_bdev; /* 相关的块文件 */
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd; /* 内存磁盘信息 */
	struct hlist_node	s_instances; /* 该文件系统的示例 */ 
	unsigned int		s_quota_types;	/* 支持的限额类型的位掩码 */
	struct quota_info	s_dquot;	/* 磁盘限额的操作 */

	struct sb_writers	s_writers;

	char s_id[32];				/* 信息的名字 */
	u8 s_uuid[16];				/* UUID：通用唯一标识符 */

	void 			*s_fs_info;	/* 文件系统的私有信息 */
	unsigned int		s_max_links;
	fmode_t			s_mode; /* 安装权限 */

	/* Granularity of c/m/atime in ns.
	   Cannot be worse than a second */
	u32		   s_time_gran;	/* 时间粒度 */

	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */

	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	char *s_subtype; /* 子类型 */

	/*
	 * Saved mount options for lazy filesystems using
	 * generic_show_options()
	 */
	char __rcu *s_options;
	const struct dentry_operations *s_d_op; /* 默认的对目录项的操作 */

	/*
	 * Saved pool identifier for cleancache (-1 means none)
	 */
	int cleancache_poolid;

	struct shrinker s_shrink;	/* per-sb shrinker handle */

	/* Number of inodes with nlink == 0 but still referenced */
	atomic_long_t s_remove_count;

	/* Being remounted read-only */
	int s_readonly_remount;

	/* AIO completions deferred from interrupt context */
	struct workqueue_struct *s_dio_done_wq;
	struct hlist_head s_pins;

	/*
	 * Keep the lru lists last in the structure so they always sit on their
	 * own individual cachelines.
	 */
	struct list_lru		s_dentry_lru ____cacheline_aligned_in_smp;
	struct list_lru		s_inode_lru ____cacheline_aligned_in_smp;
	struct rcu_head		rcu;
	struct work_struct	destroy_work;

	struct mutex		s_sync_lock;	/* 同步串行时钟 */

	/*
	 * Indicates how deep in a filesystem stack this SB is
	 */
	int s_stack_depth;	/* 此SB在在文件系统的堆栈中的深度 */

	/* s_inode_list_lock protects s_inodes */
	spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
	struct list_head	s_inodes;	/* 链接所有的节点对象 */
};


{% endhighlight %}

**超级块操作函数：**在superblock中最重要的一项是s_op,它是一个指针指向superblock的操作函数集，如下。任何文件系统可以选择性的实现这些操作，未实现的操作字段可置为NULL。当VFS调用到关联的指针为NULL，它将视操作而定执行一个通用的函数或什么都不做。
{% highlight rouge %}
struct super_operations {
	/* 在该超级块下，创建及初始化一个文件节点 */
   	struct inode *(*alloc_inode)(struct super_block *sb);
	/* 解除给定的节点 */	
	void (*destroy_inode)(struct inode *);
	/* 当某个节点内容改变时由VFS调用此函数，用它执行日志更新 */
   	void (*dirty_inode) (struct inode *, int flags);
	/* 将给定的节点写入到磁盘 */
	int (*write_inode) (struct inode *, struct writeback_control *wbc);
	/* 当某节点的最后一个引用丢失，VFS调用此函数 */
	int (*drop_inode) (struct inode *);
	/* */
	void (*evict_inode) (struct inode *);
	/* 在解除安装文件系统时由VFS调用此函数 */
	void (*put_super) (struct super_block *);
	/* 用磁盘文件系统同步文件系统元数据 */
	int (*sync_fs)(struct super_block *sb, int wait);
	int (*freeze_super) (struct super_block *);
	int (*freeze_fs) (struct super_block *);
	int (*thaw_super) (struct super_block *);
	int (*unfreeze_fs) (struct super_block *);
	/* 由VFS调用用于获取文件系统的统计数据 */
	int (*statfs) (struct dentry *, struct kstatfs *);
	/* 由VFS调用用于当需要以新的安装操作重新安装文件系统 */
	int (*remount_fs) (struct super_block *, int *, char *);
	/* VFS调用，用于中断安装操作，通常被NFS使用 */
	void (*umount_begin) (struct super_block *);

	int (*show_options)(struct seq_file *, struct dentry *);
	int (*show_devname)(struct seq_file *, struct dentry *);
	int (*show_path)(struct seq_file *, struct dentry *);
	int (*show_stats)(struct seq_file *, struct dentry *);
#ifdef CONFIG_QUOTA
	ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
	ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
	struct dquot **(*get_dquots)(struct inode *);
#endif
	int (*bdev_try_to_free_page)(struct super_block*, struct page*, gfp_t);
	long (*nr_cached_objects)(struct super_block *,
				  struct shrink_control *);
	long (*free_cached_objects)(struct super_block *,
				    struct shrink_control *);
};
{% endhighlight %}

#### 节点（linux/fs.h）
**节点：**
{% highlight rouge %}
/*
 * Keep mostly read-only and often accessed (especially for
 * the RCU path lookup and 'stat' data) fields at the beginning
 * of the 'struct inode'
 */
struct inode {
	umode_t			i_mode;
	unsigned short		i_opflags;
	kuid_t			i_uid;
	kgid_t			i_gid;
	unsigned int		i_flags;

#ifdef CONFIG_FS_POSIX_ACL
	struct posix_acl	*i_acl;
	struct posix_acl	*i_default_acl;
#endif

	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;

#ifdef CONFIG_SECURITY
	void			*i_security;
#endif

	/* Stat data, not accessed from path walking */
	unsigned long		i_ino;
	/*
	 * Filesystems may only read i_nlink directly.  They shall use the
	 * following functions for modification:
	 *
	 *    (set|clear|inc|drop)_nlink
	 *    inode_(inc|dec)_link_count
	 */
	union {
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	};
	dev_t			i_rdev;
	loff_t			i_size;
	struct timespec		i_atime;
	struct timespec		i_mtime;
	struct timespec		i_ctime;
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;
	unsigned int		i_blkbits;
	blkcnt_t		i_blocks;

#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;
#endif

	/* Misc */
	unsigned long		i_state;
	struct mutex		i_mutex;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	u64			i_version;
	atomic_t		i_count;
	atomic_t		i_dio_count;
	atomic_t		i_writecount;
#ifdef CONFIG_IMA
	atomic_t		i_readcount; /* struct files open RO */
#endif
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct block_device	*i_bdev;
		struct cdev		*i_cdev;
		char			*i_link;
	};

	__u32			i_generation;

#ifdef CONFIG_FSNOTIFY
	__u32			i_fsnotify_mask; /* all events this inode cares about */
	struct hlist_head	i_fsnotify_marks;
#endif

	void			*i_private; /* fs or device private pointer */
};
{% endhighlight %}

#### 节点操作函数
**节点操作函数：**
{% highlight rouge %}

struct inode_operations {
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
	const char * (*get_link) (struct dentry *, struct inode *, struct delayed_call *);
	int (*permission) (struct inode *, int);
	struct posix_acl * (*get_acl)(struct inode *, int);

	int (*readlink) (struct dentry *, char __user *,int);

	int (*create) (struct inode *,struct dentry *, umode_t, bool);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,umode_t);
	int (*rmdir) (struct inode *,struct dentry *);
	int (*mknod) (struct inode *,struct dentry *,umode_t,dev_t);
	int (*rename) (struct inode *, struct dentry *,
			struct inode *, struct dentry *);
	int (*rename2) (struct inode *, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
	int (*setattr) (struct dentry *, struct iattr *);
	int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
	int (*setxattr) (struct dentry *, const char *,const void *,size_t,int);
	ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
	ssize_t (*listxattr) (struct dentry *, char *, size_t);
	int (*removexattr) (struct dentry *, const char *);
	int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
		      u64 len);
	int (*update_time)(struct inode *, struct timespec *, int);
	int (*atomic_open)(struct inode *, struct dentry *,
			   struct file *, unsigned open_flag,
			   umode_t create_mode, int *opened);
	int (*tmpfile) (struct inode *, struct dentry *, umode_t);
	int (*set_acl)(struct inode *, struct posix_acl *, int);
} ____cacheline_aligned;
{% endhighlight %}
## 持续更新....