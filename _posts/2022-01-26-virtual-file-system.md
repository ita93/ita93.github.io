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

Bây giờ mình sẽ viết một kernel module mà khi insert vào nó sẽ in một chuỗi ký tự ra terminal (stdout không phải dmesg) bằng cách sử dụng file descriptor được lưu trữ trong ```file_struct```. Việc này có thể được thực hiện là do Linux kernel sử dụng terminal như một file thông thường, nó cũng hỗ trợ mở, đọc ghi qua file descriptor. Mình sẽ chỉ viết hàm init của module, và sẽ chú thích trong code:

{% highlight C linenos %}
static int nr_pid = 0; /* Tham số truyền vào của kernel module này là pid của một process */
module_param(nr_pid, int, 0644);
static int __init vfs_layer_init(void)
{
        struct files_struct *files; 
        struct file *stdout_fd = NULL;
        struct pid *bpid;
        struct task_struct *btask_struct;
        loff_t offset = 0;
        char *hello_user = "Hello userspace\n"; /* Đây là chuỗi sẽ được in ra màn hình */

        bpid = find_get_pid(nr_pid); /* Dòng này sẽ lấy pid_t object tương ứng với pid truyền vào */
        if (!bpid)
                return -EINVAL;
        rcu_read_lock();
        btask_struct = pid_task(bpid, PIDTYPE_PID); /* Lấy task_struct object của process chúng ta muốn thao tác */
        rcu_read_unlock();

        files = btask_struct->files; // Lấy ra files_struct object của process

		// Tiếp theo, chúng ta lấy ra file descriptor của stdout.
		// Trong linux thì open fd của stdout có giá trị là 1, tức là files->fdt[1], 
		// Tuy nhiên chúng ta nên sử dụng các hàm cung cấp bởi kernel để truy cập giá trị này thay vì truy cập trực tiếp
        stdout_fd = files_lookup_fd_raw(files, 1); //1 is open fd of stdout

        if (stdout_fd) {
				//Sử dụng stdout_fd để ghi ra terminal
				// Lưu ý rằng hàm __kernel_write() đã bao gồm cả code gọi tới hàm f_op->write
				// Mình gọi hàm f_op->write ở đây chỉ để show ra rằng chúng ta có thể truy cập
				// các hàm này thông qua task_struct của process.
                if (stdout_fd->f_op->write)
                        stdout_fd->f_op->write(stdout_fd,hello_user, strlen(hello_user), &offset);
                else if (stdout_fd->f_op->write_iter)
                        __kernel_write(stdout_fd, hello_user, strlen(hello_user), &offset);
        }

        return 0;
}
{% endhighlight %}

Sử dụng kernel module này như sau:
{% highlight bash linenos %}
root@oni:~# insmod vfs_layer.ko nr_pid=$(echo $$)
Hello userspace
{% endhighlight %}

File object sẽ được kết nối tới các inode (in-memory inode) thông qua dentry object,  phần tiếp theo mình sẽ viết về dentry và path lookup

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

Trong đó ```i_op``` là con trỏ tới kiểu dữ liệu ```inode_operations```, kiểu dữ liệu này là một lớp trừu tượng chứa các con trỏ hàm dùng để tương tác với dữ liệu của inode, một số trường quan trọng của ```inode_operations```:
{% highlight C linenos %}
struct inode_operations {
	/* Hàm mknod sẽ tạo ra một node mới ở mounted fs hiện tại */
	int (*mknod) (struct user_namespace *, struct inode *,struct dentry *,
		      umode_t,dev_t);
	/* Hàm create được gọi khi user muốn tạo một file thông thường */
	int (*create) (struct user_namespace *, struct inode *,struct dentry *,
		       umode_t, bool);
	/* Hàm myfs_mkdir có tác dụng tạo một thư mục mới */
	int (*mkdir) (struct user_namespace *, struct inode *,struct dentry *,
		      umode_t);
	/* Hàm lookup có tác dụng tìm kiếm */
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
	/* Hàm link*/
	int (*link) (struct dentry *,struct inode *,struct dentry *);
}
{% endhighlight %}

Một đối tượng in-memory inode mới được tạo ra bằng cách sử dụng hàm ```new_inode()```, hàm này sẽ yêu cầu bộ nhớ cho Inode mới, và thêm inode này vào cuối list ```s_inodes``` của super block được truyền vào:
{% highlight C linenos %}
struct inode *new_inode(struct super_block *sb)
{
	struct inode *inode;

	spin_lock_prefetch(&sb->s_inode_list_lock);

	inode = new_inode_pseudo(sb); /* Gọi tới inode_alloc() để yêu cầu vùng nhớ cho inode mới */

	if (inode) /* Nếu việc cấp phát diễn ra thành công thì thêm inode vào list s_inodes */
		inode_sb_list_add(inode);
	return inode;
}
{% endhighlight %}

Hàm ```inode_alloc()``` sẽ sử dụng con trỏ hàm từ ```struct super_operations``` để cấp phát inode mới, ví dụ với ext4 fs thì con trỏ hàm này sẽ trỏ tới hàm:
{% highlight C %}
static struct inode *ext4_alloc_inode(struct super_block *sb)
{% endhighlight %}

Như đã nói ở trên, thì ```super_block``` chứa các thông tin về một mounted fs, các super block sẽ được tạo ra khi người dùng thực hiện mount một phân vùng mới thông qua hàm mount. VFS sử dụng các hàm fill_super để load các thông tin về fs vừa được mount vào trong ```super_block```. Ngoài ra VFS cũng hỗ trợ việc truyền vào các tùy chọn cho một mount fs, (ví dụ: ``` mount -t iso9660 -o ro /dev/cdrom /mnt```) hay việc giải phóng các tài nguyên bộ nhớ liên quan đến một mounted fs khi chúng ta umount. Tất cả các nhiệm vụ này được VFS thực hiện thông qua một cấu trúc dữ liệu tên là ```fs_context_operations```.
#### Subsection: [File system context](https://elixir.bootlin.com/linux/v5.16/source/Documentation/filesystems/mount_api.rst) (kernel 5.4)
Trước đây, việc mount một fs mới được thực hiện thông qua con trỏ hàm ```mount()``` của cấu trúc dữ liệu ```struct file_system_type```, tuy nhiên kể từ phiên bản kernel <i>5.4</i>, linux kernel đã giới thiệu một cơ chế mới có tên là <i>file system context</i>, cơ chế này cung cấp cho VFS khả năng tham số hóa qua trình khởi tạo/tìm kiếm/tái cấu hình super block.
Qúa trình tạo một mount point mới của VFS sẽ được thực hiện qua các bước sau:
- Bước 1: Tạo ra một file system context. 
- Bước 2: Phân tích các tham số đầu vào và gắn nó vào fs context. 
- Bước 3: Kiểm tra tính hợp lệ.
- Bước 4: Tạo mới hoặc load một super block và root inode.
- Bước 5: Thực hiện việc mount
- Bước 6: Trả về một errorf (nếu có lỗi)
- Bước 7: Giải phóng fs context.

Việc tạo và tái cấu trúc một superblock được quản lý bởi một fs context, VFS sử dụng cấu trúc dữ liệu ```struct fs_context``` để lưu giữ các thông tin về một fs context.
{% highlight C linenos %}
struct fs_context {
	/* ops là các thao tác có thể được thực hiện trên một fs context, đây là một trường quan trọng và bắt buộc phải được gán giá trị tại thời điểm khởi tạo một fs context */
	const struct fs_context_operations *ops;
	/* Một con trỏ tới fs type (chủ của fs context này), trường này giúp cho việc thay đổi các giá trị của fs type đơn giản hơn */
	struct file_system_type *fs_type;
	/* Chứa private data, cấu trúc dữ liệu có thể được định nghĩa bởi developer của fs type */
	void			*fs_private;
	/* Root của mountable tree */
	struct dentry		*root;
	/* Namespace */
	struct user_namespace	*user_ns;
	struct net		*net_ns;
	/* Credentials */
	const struct cred	*cred;
	/* Source, chẳng hạn như /dev/sda1 hoặc host:/path */
	char			*source;
	char			*subtype;
	/* Security data của super block */
	void			*security;
	/* Dùng để phân biệt các supber block */
	void			*s_fs_info;
	unsigned int		sb_flags;
	unsigned int		sb_flags_mask;
	unsigned int		s_iflags;
	unsigned int		lsm_flags;
	/* FS_CONTEXT_FOR_MOUNT, FS_CONTEXT_FOR_SUBMOUNT hoặc FS_CONTEXT_FOR_RECONFIGURE */
	enum fs_context_purpose	purpose:8;
	/* Cut... */
};
{% endhighlight %}
Linux kernel sử dụng hàm ```alloc_fs_context()``` để tạo mới một đối tượng thuộc kiểu dữ liệu này. Trường ```ops``` của cấu trúc dữ liệu này là một con trỏ tới một đối tượng ```fs_context_operations```, đối tượng này chứa các con trỏ hàm được sử dụng ở các giai đoạn khác nhau trong cơ chế mount của fs context.
{% highlight C linenos %}
struct fs_context_operations {
	/* Cleanup fs context */
	void (*free)(struct fs_context *fc); 
	/* Duplicate the fs-private data. */
	int (*dup)(struct fs_context *fc, struct fs_context *src_fc);
	/* Parse options */
	int (*parse_param)(struct fs_context *fc, struct fs_parameter *param);
	int (*parse_monolithic)(struct fs_context *fc, void *data);
	/* Sẽ gọi đến hàm fill_super() dùng để tạo mountable và superblock */
	int (*get_tree)(struct fs_context *fc);
	/* Reconfigure */
	int (*reconfigure)(struct fs_context *fc);
};
{% endhighlight %}
VFS cung cấp một số hàm để sử dụng cho mục đích tạo super block và root node (Được gọi từ ```get_tree()``` của fs context operations):
{% highlight C %}
int get_tree_nodev(struct fs_context *fc,
		  int (*fill_super)(struct super_block *sb,
				    struct fs_context *fc));
int get_tree_single(struct fs_context *fc,
		  int (*fill_super)(struct super_block *sb,
				    struct fs_context *fc));
int get_tree_single_reconf(struct fs_context *fc,
		  int (*fill_super)(struct super_block *sb,
				    struct fs_context *fc));
int get_tree_keyed(struct fs_context *fc,
		  int (*fill_super)(struct super_block *sb,
				    struct fs_context *fc),
		void *key);
{% endhighlight %}
Cả 4 hàm trên đều là wrapper của hàm ```vfs_get_super()```. ```vfs_get_super()``` có tác dụng tìm kiếm hoặc khởi tạo mới một super block nếu nó chưa tồn tại. Hàm này sẽ sử dụng con trỏ hàm ```fill_super``` được truyền vào để tạo một super block mới nếu cần thiết. Quá trình tìm kiếm được điều khiển bởi @keyring, @keyring có thể có các giá trị khác nhau như sau:
- ```vfs_get_single_super``` Chỉ một super block của fs type này có thể tồn tại trong hệ thống.
- ```vfs_get_keyed_super ``` Các super block của fs type này cần có giá trị key khác nhau (key được lưu trong s_fs_info),
- ```vfs_get_idependent_super``` Có thể tồn tại nhiều super block của fs type này và không cần key.
<br><br>
{% highlight C %}
int (*parse_param)(struct fs_context *fc, struct fs_parameter *param);
{% endhighlight %}
sử dụng ```struct fs_parameter_spec``` và hàm ```fs_parse()``` để phân tích các tham số truyền vào. Mỗi đối tượng ```struct fs_parameter_spec``` giống như một entry trong từ điển, nó có một key, một value và có type, ví dụ:
{% highlight C linenos %}
struct fs_parameter_spec {
	const char		*name;
	fs_param_type		*type;	/* The desired parameter type */
	u8			opt;	/* Option number (returned by fs_parse()) */
	unsigned short		flags;
	const void		*data;
};

static const struct fs_parameter_spec proc_fs_parameters[] = {
	fsparam_u32("gid",	Opt_gid),
	fsparam_string("hidepid",	Opt_hidepid),
	fsparam_string("subset",	Opt_subset),
	{}
};
{% endhighlight %}
<br>
Tùy thuộc vào kiểu dữ liệu khai báo mà VFS sẽ sử dụng hàm parser phù hợp, danh sách các hàm/kiểu dữ liệu được hỗ trợ bởi VFS có thể xem ở [đây](https://elixir.bootlin.com/linux/v5.16/source/include/linux/fs_parser.h#L119)

<b>VFS schema</b><br><br>
![Linux management structures](/images/vfs/vfs.png)
