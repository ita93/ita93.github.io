---
layout: post

category: Linux device driver

comments: true
---

# Linux Virtual File System
Virtual file system(VFS) là cơ chế Linux sử dụng để cho phép các chương trình chạy ở userspace tương tác với hệ thống file của máy, cũng như cung cấp một giao diện đồng nhất cho phép các chương trình này có thể thực hiện các thao tác trên các loại File system khác nhau mà không cần phải thay đổi mã nguồn. 
Nhờ có VFS mà Linux có thể sử dụng được nhiều loại file system khác nhau trên cùng một hệ thống:
- Disk-based file system : Lưu trữ dữ liệu trên các thiết bị lưu trữ vật lý (ssd, hdd), các file system type tiêu biểu của loại này bao gồm: ext3, ext4, NTFS, FAT32,...
- Network filesystems: Truy cập dữ liệu từ xa qua mạng, chẳng hạn như NFS hay CIFS.
- Special filesystems: chẳng hạn như procfs, debugfs, sysfs.
![Linux FS layout](/images/vfs/vfs_layout.png)

Mặc dù Linux kernel được việt bằng C, nhưng VFS được xây dựng dựa trên các lý thuyết hướng đối tượng. VFS bao gồm nhiều object khác nhau cấu tạo nên:
- ``struct task_struct``: Thông tin về một process.
- ``struct file_struct``: Lưu giữ thông tin về các opened file của một process.
- ``struct file``: Đây là đối tượng lưu giữ các thông tin về mối liên kết giữa một process và một open file. Đối tượng này được lưu trữ trong Main Memory.
- ``struct dentry``: Lưu giữ thông tin về mối liên hệ giữa directory entry và file. Các FS type khác nhau sẽ có cách lưu trữ thông tin này trong thiết bị lưu trữ vật lý theo cách riêng của nó.
- ``Inode``: Lưu giữ siêu dữ liệu (meta data) của file. Đối với các FS vật lý thông thường, những đối tượng này sẽ tương đương với các File Control block được lưu trữ trên đĩa. Mỗi inode sẽ có một inode number được sử dụng như định danh của file.
- ``struct address_space``: Sử dụng bởi page cache, dùng để map các page của một file (địa chỉ trong bộ nhớ chính) với các disk block (địa chỉ trên đĩa/thiết bị lưu trữ).
- ``superblock``: Lưu thông tin về mounted file system.
![Data structures relationship](/images/vfs/ds_relationship.PNG)

### 1. Các cấu trúc dữ liệu VFS liên kết với process.
Ai tìm hiểu về linux kernel đều biết rằng Linux kernel sử dụng  ``struct task_struct`` để biểu diễn một process. Trong struct này có hai trường dùng để mô tả mối quan hệ giữa một process với các file:
{% highlight C linenos %}
struct task_struct {
  /* cut */
  struct fs_struct *fs;
  struct file_struct *files;
  /* cut */
};
{% endhighlight %}
<!--
{% highlight C linenos %}
struct fs_struct {
	int users;
	spinlock_t lock;
	seqcount_spinlock_t seq;
	int umask;
	int in_exec;
	struct path root, pwd;
} __randomize_layout;
{% endhighlight %}
-->

<figure>
  <img src="{{ site.url }}/images/vfs/process_view_fs.png" alt="Process's view of file system" style="width:50%">
  <figcaption>File system objects và process</figcaption>
</figure>
Trường ```fs``` là một đối tượng kiểu ```fs_struct``` chứa các filesystem information.
Trong cấu trúc này thì, users là số lượng process đang sử dụng đối tượng ```fs_struct``` này, ```root``` và ```pwd``` là hai con trỏ thuộc kiểu ```struct path*``` lần lượt là đường dẫn tới thư mục root và thư mục hiện thời của process.
Trường ```fs``` này được gán khi một process mới được tạo ra thông qua hàm [copy_fs()](https://elixir.bootlin.com/linux/v5.16/source/kernel/fork.c#L1512). Trong trường hợp cờ CLONE_FS được set, ```copy_fs``` sẽ khởi tạo vùng nhớ mới, mà sẽ tăng giá trị của counter ```users``` thêm một đơn vị, process mới được tạo ra sẽ dùng chung ```fs``` của process cha. Trong trường hợp CLONE_FS không được set, thì một vùng nhớ mới sẽ được cấp phát để tạo ra một đối tượng ```struct fs_struct``` mới tách biệt với process cha. Do các process thông thường đều là con cháu của process init nên ```fs->root``` của các process này đều giống của process init. Trường hợp chúng ta sử dụng [chroot](https://man7.org/linux/man-pages/man2/chroot.2.html) thì ```fs->root``` sẽ mang một giá trị khác.
{% highlight bash %}
# Call stack tới copy_fs()
#0  copy_fs (tsk=<optimized out>, clone_flags=18874368) at kernel/fork.c:1515
#1  copy_process (pid=pid@entry=0x0 <fixed_percpu_data>, trace=trace@entry=0, node=node@entry=-1, args=args@entry=0xffffc900054e7e58) at kernel/fork.c:2155
#2  0xffffffff81454ca7 in kernel_clone (args=args@entry=0xffffc900054e7e58) at kernel/fork.c:2555
#3  0xffffffff81455ed8 in __do_sys_clone (clone_flags=18874385, newsp=0, parent_tidptr=0x0 <fixed_percpu_data>, child_tidptr=0x7fc7398aa8d0, tls=0) at kernel/fork.c:2672
#4  0xffffffff89616c75 in do_syscall_x64 (nr=<optimized out>, regs=0xffffc900054e7f58) at arch/x86/entry/common.c:50
#5  do_syscall_64 (regs=0xffffc900054e7f58, nr=<optimized out>) at arch/x86/entry/common.c:80
#6  0xffffffff89800068 in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:113
{% endhighlight %}

Trường ```files``` là một đối tượng kiểu ```struct files_struct``` chứa thông tin về các opened files của process. [struct files_struct](https://elixir.bootlin.com/linux/v5.16/C/ident/files_struct) chứa một mảng các opened file descriptor của process hiện tại, ngoài ra ```struct files_struct``` cũng có một trường ```count``` để đếm số process đang chia sẻ nó, counter này sẽ được tăng giá trị nếu như process cha set cờ CLONE_FILES khi nó tạo ra process con.
```struct files_struct``` cũng chứa một ```fdtable``` instance, và một con trỏ tới instance này, các trường trong ```struct fdtable``` về cơ bản là các pointer tới các trường của ```files_struct``` chứa nó, như trong hình sau
<figure>
  <img src="{{ site.url }}/images/vfs/files_struct.jpg" alt="Files_struct và fdtable">
  <figcaption>Files_struct và fdtable</figcaption>
</figure>
- ```fdtable->max_fds``` là số opened file descriptors tối đa của process.
- ```fdtable->fd``` là một mảng các con trỏ tới các đối tượng ```struct file```. Giá trị file descriptor trả về khi chúng ta gọi hàm ```open()``` ở user-space chính là INDEX của mảng này.
- ```fdtable->open_fds``` là con trỏ tới một mảng đánh dấu, mục đích của mảng này là để kiểm tra xem giá trị file descriptor có khả dụng (0) hay không (1). Giá trị ```next_fd``` của ```files_struct``` chính là vị trí tiếp theo chúng ta nên kiểm tra trong mảng này nếu như process yêu cầu một opened file descriptor mới.
- ```fdtable->close_on_exec``` cũng là một mảng đánh dấu, giá trị 1 nghĩa là file descriptor nên được giải phóng khi ```exec()``` được gọi tới.
- ```files_struct->fd_array``` có kích thước ban đầu khi khởi tạo process là ```NR_OPEN_DEFAULT```
<b>Note:</b> <i>Mục đích của việc tạo ra ```struct fdtable``` để support RCU.</i><br>
Ban đầu (ở task init), thì ```files_struct->fdt``` sẽ trỏ đến ```files_struct->fdtab```, tuy nhiên nếu process mở quá nhiều files và ```fd_array``` không còn khả năng lưu trữ chúng nữa thì một đối tượng ```fdtable``` mới sẽ được tạo ra và nó ```fdt``` sẽ trỏ tới đối tượng mới này.

Ví dụ về một call stack khi process muốn tạo một opened file descriptor mới:
{% highlight bash %}
#0  expand_files (files=files@entry=0xffff888011dd4b80, nr=nr@entry=0) at fs/file.c:201
#1  0xffffffff81dd0cec in alloc_fd (start=start@entry=0, end=1024, flags=flags@entry=0) at fs/file.c:495
#2  0xffffffff81dd127b in __get_unused_fd_flags (nofile=<optimized out>, flags=0) at fs/file.c:530
#3  0xffffffff8efed163 in init_dup (file=file@entry=0xffff888049dbc040) at fs/init.c:264
{% endhighlight %}

Hàm ```alloc_fd()``` sẽ tìm một file descriptor mới (giá trị index nhỏ nhất của mảng fdt còn khả dụng), ngoài ra nó sẽ gọi tới hàm ```expand_files()``` để thực hiện việc mở rộng ```fdtable``` nếu cần thiết. (line 8)
{% highlight C linenos %}
/* Cut */
if (fd < files->next_fd)
	fd = files->next_fd;

if (fd < fdt->max_fds)
	fd = find_next_fd(fdt, fd);
/* Cut */
error = expand_files(files, fd);
/* Cut */
{% endhighlight %}

Hàm ```expand_files()``` sẽ thực hiện việc mở rộng ```fdtable``` trong trường hợp file descriptor mới được cấp phát có giá trị lớn hơn ```max_fds``` hiện tại, nó sẽ gọi đến ```expand_fdtable()```. Hàm ```expand_fdtable()``` sẽ cấp phát một vùng nhớ cho fdtable và fd_array mới(? Đây là function comment trong source code, nhưng có vẻ không đúng).
{% highlight C linenos %}
static int expand_fdtable(struct files_struct *files, unsigned int nr)
{
  struct fdtable *new_fdt, *cur_fdt;
  spin_unlock(&files->file_lock);
  new_fdt = alloc_fdtable(nr); /* Cấp phát fdtable mới có thể chứa được nr file descriptor*/
  /*Cut*/
  cur_fdt = files_fdtable(files); 
	BUG_ON(nr < cur_fdt->max_fds);
	copy_fdtable(new_fdt, cur_fdt); /* Sao chép dữ liệu từ fdtable hiện tới tới fdtable mới cấp phát */
	rcu_assign_pointer(files->fdt, new_fdt); /* Gán giá trị fdt của files_struct tới fdtable mới cấp phát */
	if (cur_fdt != &files->fdtab)
		call_rcu(&cur_fdt->rcu, free_fdtable_rcu); /* Giải phóng fdtable cũ */
	/* coupled with smp_rmb() in fd_install() */
	smp_wmb();
	return 1;
}
{% endhighlight %}
Kể từ sau lần gọi đầu tiên đến ```expand_fdtable()``` thì ```fdtable->fd``` không còn trỏ tới ```files_struct->fd_array``` nữa.
{% highlight C linenos %}
struct files_struct {
	/* cut */
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
	/* cut */
}
{% endhighlight %}

### 2. Struct [file](https://elixir.bootlin.com/linux/v5.16/source/include/linux/fs.h#L962) - Đối tượng biểu diễn một opened file, và phương thức process tương tác với file đó.
Mỗi đồi tượng ```struct file``` là một phần tử trong open file table, một đối tượng ```struct file``` mới sẽ được thêm vào table này khi hàm ```open()``` được gọi, nhiều ```file``` object có thể trỏ tới cùng một file vật lý. Đây chính là những object được trỏ tới bởi file descriptor table (fdtable->fd và fd_array).

Các trường quan trọng của file object:
- ```struct llist_node fu_llist``` 
- ```struct path f_path```
- ```struct fmode_t f_mode```
- ```loff_t f_pos```
- ```const struct file_operations *f_op``` [Đã đề cập trong bài về character device driver]({{ site.baseurl }}{% link _posts/2017-12-18-character-device.md %})

<i>file pointer</i> ```f_pos``` sẽ cho biết vị trị hiện tại của file, tức là vị trí trong file mà thao tác tiếp theo của ```file_operations``` sẽ tác động tới. <br>
File object được cấp phát bằng hàm [__alloc_file()](https://elixir.bootlin.com/linux/v5.16/C/ident/__alloc_file), và các file object đều nằm trong slab cache ```filp_cache```
{% highlight C linenos %}
static struct file *__alloc_file(int flags, const struct cred *cred)
{
	struct file *f;
	int error;

	f = kmem_cache_zalloc(filp_cachep, GFP_KERNEL);
	/* Cut */
{% endhighlight %}

File object sẽ được kết nối tới các inode (disk file descriptor) thông qua dentry object. 

### 3. Dentry và path lookup
{% highlight C linenos %}
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_spinlock_t d_seq;	/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory dentry object */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	union {
		struct list_head d_lru;		/* LRU list */
		wait_queue_head_t *d_wait;	/* in-lookup ones only */
	};
	struct list_head d_child;	/* list of children from the parent directory (our siblings) */
	struct list_head d_subdirs;	/* list of our children (files and subdirectories)  */
	/*
	 * d_alias and d_rcu can share memory
	 */
	union {
		struct hlist_node d_alias;	/* inode alias list */
		struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
	 	struct rcu_head d_rcu;
	} d_u;
} __randomize_layout;
{% endhighlight %}
Dentry của file object được wrap trong ```struct path f_path```.<br>
<b>Dentry</b> giúp cho Linux có thể sử dụng thưc mục như thể nó là một file bình thường, ngoài ra nó cũng giúp đơn giản hóa việc phân giải đường dẫn (path resolving). Trong Linux thì thư mục đơn giản là một file có nội dung là danh sách các file con của thư mục đó: 
{% highlight C linenos %}
# Nội dung của /tmp khi mở bằng ứng dụng vi
/tmp/
▸ cscope.1018443/
▸ nvimYeSwDV/
▸ ssh-0DuqRUmLe20F/
▸ ssh-4YzTJFipav60/
▸ ssh-7hZUUTpWcEZr/
▸ ssh-CpraiX7DPMpJ/
test1
test2
test3
{% endhighlight %}
Khi một file object muốn thực hiện thao tác với dữ liệu, linux kernel cần phân giải đường dẫn thông thường (ví dụ: /usr/bin/gdb) thành inode number tương ứng. Việc này được thực hiện thông qua các dentry, lưu giữ trong trong main memory để tăng tốc độ phân truy cập. <br>
Ví dụ, đối với việc truy cập <i>/usr/bin/gdb</i>, linux kernel sẽ tạo ra ```dentry``` cho : <b>/</b>, <b>usr</b>, <b>bin</b>, <b>gdb</b>, và mỗi ```dentry``` này sẽ được kết nói với một inode nhất định thông qua trường ```d_inode```.
Thông qua các trường ```d_parent```, ```d_child```, ```d_subdirs```, các dentry objects sẽ được liên với nhau để tạo ra một cây thư mục, có cấu trúc cây giống như những gì chúng ta thường thấy qua các file explorer.
<figure>
  <img src="{{ site.url }}/images/vfs/vfs-dentry-inode.jpg" alt="Files_struct và fdtable">
  <figcaption>"Cây" thư mục</figcaption>
</figure>
<br>
Mỗi ```dentry``` có thể tồn tại ở một trong bốn trạng thái sau đây:
- <b>free</b> : Không chứa bất kỳ thông tin nào quan trọng, nó không được sử dụng bởi VFS.
- <b>unused</b> : Đã từng được sử dụng, tuy nhiên hiện tại nó không còn được sử dụng nữa (```d_lockref.count``` bằng không), nhưng ```d_node``` vẫn trỏ tới inode.
- <b>used</b> : Sử dụng bởi kernel, ```d_lockref.count``` > 0 và ```d_inode``` trỏ tới inode.
- <b>empty</b> : ```d_inode``` NULL, dentry này vẫn có thể được sử dụng trong các thao tác tìm kiếm trong tương lai, tuy nhiên inode liên kết với dentry này đã bị xóa, hoặc dentry được tạo ra để phân giải một file vật lý không tồn tại.

Trường ```d_op``` của ```struct dentry``` là chứa các con trỏ hàm giúp thao tác với ```dentry``` object, Một số hàm được ```d_op``` hỗ trợ bao gồm:
- ```d_revalidate(dentry, nameidata)``` trước khi phân giải đường dẫn tới file, hàm này sẽ được gọi để kiểm tra tính hợp lệ cảu ```dentry``` object; đa số các disk fs không hỗ trợ hàm này, tuy nhiên hầu hết các net fs đều hỗ trợ.
- ```d_delete(dentry)```  hàm này được gọi khi dentry reference counter chuyển về giá trị 0.
- ```d_release(dentry)``` VFS gọi hàm này để giải phóng dentry.
- ```d_iput(dentry, ino)``` gọi khi dentry object bị xóa mất inode (chuyển sang trạng thái empty)
- ```d_hash(dentry, name)``` Tạo hash value cho dentry.
- ```d_compare(dir, name1, name2)``` So sánh hai file name.
Do việc tạo các đối tượng ```dentry``` tương ứng từ directory entry tốn thời gian đáng kể, nên VFS sử dụng dentry cache (dcache) để lưu trữ một số đối tượng ```dentry```không sử dụng trong bộ nhớ chính, do những đối tượng này có thể cần tới về sau. 
{% highlight C %}
static struct kmem_cache *dentry_cache __read_mostly;
{% endhighlight %}
Các ```dentry``` object đều được link tới bởi các entry trong [dentry_hashtable](https://elixir.bootlin.com/linux/v5.16/source/fs/dcache.c#L101) thông qua trường ```d_hash```, bảng băm này sẽ tăng tốc độ tìm kiếm dentry. ```dentry_hashtable``` có kiểu ```struct hlist_bl_head```, về cơ bản nó là một mảng các ```struct hlist_bl_node```, các node này là node đầu tiên trong một danh sách liên kết các ```dentry``` có cùng hash value. Sức chứa của ```dentry_hashtable``` phụ thuộc vào dung lượng RAM, 
{% highlight bash linenos %}
# 2GB RAM system
root@syzkaller:~# dmesg |grep "Dentry cache"
[    0.717333] Dentry cache hash table entries: 262144 (order: 9, 2097152 bytes, vmalloc)
{% endhighlight %}
<figure>
  <img src="{{ site.url }}/images/vfs/dcache.png" alt="dentry_hashtable">
  <figcaption>Dentry Hashtable</figcaption>
</figure>
<br>
{% highlight C linenos %}
# Simpelifed code
static unsigned int d_hash_shift __read_mostly;
static struct hlist_bl_head *dentry_hashtable __read_mostly;
static inline struct hlist_bl_head *d_hash(unsigned int hash)
{
	return dentry_hashtable + (hash >> d_hash_shift);
}

# Block code thêm một dentry vào hash table
{
	struct hlist_bl_head *b = d_hash(entry->d_name.hash);
	hlist_bl_lock(b);
	hlist_bl_add_head_rcu(&entry->d_hash, b);
	hlist_bl_unlock(b);
}
{% endhighlight %}
Ngoài ra, tất cả các <i>unused dentry</i> đều được liên kết với nhau trong một double-linked [LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) list thông qua trường ```d_lru```, hầu hết các dentry nằm trong LRU này đều góp mặt trong ```dentry_hashtable```.<br>

Việc phân giải tên sẽ được thực hiện thông qua hàm ```link_path_walk()```, hàm này sẽ convert một pathname thành inode number, nó được gọi tới bởi các function như : ```open()```, ```stat()```,...
{% highlight bash linenos %}
# Stacktrace của system call tới sys_open()
(gdb) bt
#0  0xffffffff81d93d90 in link_path_walk (nd=<optimized out>, name=<optimized out>) at fs/namei.c:2270
#1  path_openat (nd=nd@entry=0xffffc90001c77c30, op=op@entry=0xffffc90001c77df0, flags=flags@entry=65) at fs/namei.c:3605
#2  0xffffffff81d98d41 in do_filp_open (dfd=dfd@entry=-100, pathname=pathname@entry=0xffff88801992a480, op=op@entry=0xffffc90001c77df0) at fs/namei.c:3636
#3  0xffffffff81d4116b in do_sys_openat2 (dfd=dfd@entry=-100, filename=filename@entry=0x7f56072b6466 "/proc/self/oom_score_adj", how=how@entry=0xffffc90001c77ea8) at fs/open.c:1214
#4  0xffffffff81d45bd3 in do_sys_open (dfd=-100, filename=0x7f56072b6466 "/proc/self/oom_score_adj", flags=<optimized out>, mode=<optimized out>) at fs/open.c:1230
{% endhighlight %}
Hàm ```link_path_walk()``` nhận vào hai tham số là path name và [nameidata](https://elixir.bootlin.com/linux/latest/source/fs/namei.c#L563) object
VFS sẽ bắt đầu việc phân giải pathname từ ```nameidata->path```, phụ thuộc vào giá trị của trường ```nameidata->dfd``` mà vị trí bắt đầu tìm kiếm có thể là dentry lien kết với ```root``` hoặc ```pwd```. Quá trình phân giải path name được thực hiện như sau: 
- Tìm kiếm địa chỉ inode tương ứng với element gần nhất (từ trái qua)
- Kiểm trả xem process có quyền thực thi không (execute permission).
- Lấy element tiếp theo từ path name.
- Kiểm tra các trường hợp đặc biệt (```.``` hoặc ```..```)
- Tìm kiếm dcache entry cho dentry object tương ứng với element gần nhất, nếu việc tìm kiếm trong dcache thất bại thì VFS phải load dentry object từ đĩa và lưu vào dcache.
kiểm tra xem element hiện tại có phải là mount point không? Nếu có thể inode hiện tại sẽ thay đổi thành root inode của mounted file system.
- Nếu element đang xử lý không phải là element cuối cùng của pathname thì tiếp tục thực hiện các bước trên cho element tiếp theo.

### 4. Inode và Super block
Note: Cho ngắn gọn dễ nghe thì tui tự quy ước file vật lý nghĩa là file nằm trong disk.
Superblock là block đầu tiên của một mounted file system, nó chứa mounted file system's metadata, chẳng hạn như số blocks, số inodes, etc,... <br>
Inode là một thành phần quan trọng của các UNIX-style file system (được trên thiết bị lưu trữ vật lý - on-disk inode), đồng thời cũng là một thành phần quan trọng của VFS (in-memory inode). Mỗi inode là một định danh về một file vật lý cũng như các thông tin về file đó (uid, gid, permission, etc), mặc dù vậy, inode không có thông tin về tên file, tên file là thông tin thuộc về ```struct dentry``` object liên kết với ```inode``` này. Tên file chỉ là một label để người dùng có thể dễ dàng ghi nhớ và nhận diện, nó có thể thay đổi tùy thuộc vào thao tác của người dùng, tuy nhiên inode và inode number của một file sẽ không bao giờ thay đổi. Khi một process gọi hàm ```open()``` thì nó sẽ tạo ra một ```file``` object liên kết với ```inode``` object, tại cùng một thời điểm có thể có nhiều ```file``` object cùng tham chiếu tới một ```inode```.
Tương tự như đối với ```dentry```, VFS cũng sử dụng một bảng băm tên là ```inode_hastable``` và một LRU list để truy xuất và tái sử dụng inode một cách hiệu quả. LRU list này được lưu trữ bởi <i>in-memory superblock</i>:
{% highlight C %}
list_lru_add(&inode->i_sb->s_inode_lru, &inode->i_lru)
{% endhighlight %}
Một số trường quan trọng của ```struct inode```
{% highlight C linenos %}
struct inode {
	struct super_block *i_sb; /* pointer tới superblock sử hữu inode này */
	dev_t i_rdev; /* Device number chứa mounted fs*/
	unsigned long i_ino; /* Inode number trong disk inode table*/
	u8 i_blkbits;
	umode_t i_mode; /*file type: regular, directory, named pipe, etc...*/
	kuid_t i_uid;
	kgid_t i_gid;
	loff_t i_size;
	struct timespec i_mtime, i_atime, i_ctime;
	i_nlink;
	blkcnt_t i_blocks;
	const struct inode_operations *i_op;
	const struct file_operations *i_fop;	
	atomic_t i_count;
}
{% endhighlight %}

<b>VFS schema</b><br><br>
![Linux management structures](/images/vfs/vfs.png)
