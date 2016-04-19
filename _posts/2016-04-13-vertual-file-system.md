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

## 核心数据结构

先看一下典型的linux文件系统EXT2在磁盘中文件的组成形式：

**图1：ext2磁盘文件系统**

![](/img/vfs/fsondisk.jpg)

#### 超级块(linux/fs.h)

**超级块对象：**该数据结构包含文件系统的安装信息，超级块一般安装在磁盘分区的引导块紧邻之后，在文件系统安装时起到作用。
{% highlight rouge %}
struct super_block {
	struct list_head	s_list;		/* Keep this first */
	dev_t			s_dev;		/* search index; _not_ kdev_t */
	unsigned char		s_blocksize_bits;
	unsigned long		s_blocksize;
	loff_t			s_maxbytes;	/* Max file size */
	struct file_system_type	*s_type;
	const struct super_operations	*s_op;
	const struct dquot_operations	*dq_op;
	const struct quotactl_ops	*s_qcop;
	const struct export_operations *s_export_op;
	unsigned long		s_flags;
	unsigned long		s_iflags;	/* internal SB_I_* flags */
	unsigned long		s_magic;
	struct dentry		*s_root;
	struct rw_semaphore	s_umount;
	int			s_count;
	atomic_t		s_active;
#ifdef CONFIG_SECURITY
	void                    *s_security;
#endif
	const struct xattr_handler **s_xattr;

	struct hlist_bl_head	s_anon;		/* anonymous dentries for (nfs) exporting */
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
	struct block_device	*s_bdev;
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd;
	struct hlist_node	s_instances;
	unsigned int		s_quota_types;	/* Bitmask of supported quota types */
	struct quota_info	s_dquot;	/* Diskquota specific options */

	struct sb_writers	s_writers;

	char s_id[32];				/* Informational name */
	u8 s_uuid[16];				/* UUID */

	void 			*s_fs_info;	/* Filesystem private info */
	unsigned int		s_max_links;
	fmode_t			s_mode;

	/* Granularity of c/m/atime in ns.
	   Cannot be worse than a second */
	u32		   s_time_gran;

	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */

	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	char *s_subtype;

	/*
	 * Saved mount options for lazy filesystems using
	 * generic_show_options()
	 */
	char __rcu *s_options;
	const struct dentry_operations *s_d_op; /* default d_op for dentries */

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

	struct mutex		s_sync_lock;	/* sync serialisation lock */

	/*
	 * Indicates how deep in a filesystem stack this SB is
	 */
	int s_stack_depth;

	/* s_inode_list_lock protects s_inodes */
	spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
	struct list_head	s_inodes;	/* all inodes */
};


{% endhighlight %}

**超级块操作函数：**
{% highlight rouge %}

struct super_operations {
   	struct inode *(*alloc_inode)(struct super_block *sb);
	void (*destroy_inode)(struct inode *);

   	void (*dirty_inode) (struct inode *, int flags);
	int (*write_inode) (struct inode *, struct writeback_control *wbc);
	int (*drop_inode) (struct inode *);
	void (*evict_inode) (struct inode *);
	void (*put_super) (struct super_block *);
	int (*sync_fs)(struct super_block *sb, int wait);
	int (*freeze_super) (struct super_block *);
	int (*freeze_fs) (struct super_block *);
	int (*thaw_super) (struct super_block *);
	int (*unfreeze_fs) (struct super_block *);
	int (*statfs) (struct dentry *, struct kstatfs *);
	int (*remount_fs) (struct super_block *, int *, char *);
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

## 持续更新....