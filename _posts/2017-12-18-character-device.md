# Character Device Driver(DRAFT Ver).

Character Device (chardev) là các device được truy cập như một luồng nhị phân (byte stream - tương tự như các file trong máy tính), và character device
driver có nhiệm vụ thực hiện những thao tác đọc ghi này. Đối với một character device driver, ít nhất 4 system call: open, close read và write cần được implement để chardev có thể hoạt động một cách bình thường. 

Một số ví dụ về character device là: Serial console và console.

Do Linux về cơ bản là file, file và file nên các character device cũng không phải là ngoại lệ, chúng được truy cập bằng các file system node (trong /dev directory). Tuy nhiên, không giống như các file thông thường khác (có thể truy cập tiến, lùi tùy ý), các file dev này chỉ được truy cập theo một cách tuần tự. 

Khi dùng command ls -l /dev, các device file được liệt kê, các file có chữ <span style="color:blue">c</span> ở đầu là các device file của chardev.

```python
crw-r--r--    1 root     root        5,   1 Dec 27 03:45 console
crw-rw-rw-    1 root     root        1,   7 Jan  1  1970 full
crw-r--r--    1 root     root       10, 229 Dec 27 03:06 fuse
crw-r--r--    1 root     root        1,  11 Jan  1  1970 kmsg
crw-r--r--    1 root     root        1,   1 Jan  1  1970 mem
brw-r--r--    1 root     root       31,   0 Jan  1  1970 mtdblock0
brw-r--r--    1 root     root       31,   1 Jan  1  1970 mtdblock1
brw-r--r--    1 root     root       31,   2 Jan  1  1970 mtdblock2
brw-r--r--    1 root     root       31,   3 Jan  1  1970 mtdblock3
brw-r--r--    1 root     root       31,   4 Jan  1  1970 mtdblock4
brw-r--r--    1 root     root       31,   5 Jan  1  1970 mtdblock5
brw-r--r--    1 root     root       31,   6 Jan  1  1970 mtdblock6
brw-r--r--    1 root     root       31,   7 Jan  1  1970 mtdblock7
crw-rw-rw-    1 root     root        1,   3 Jan  1  1970 null
crw-r--r--    1 root     root        1,   4 Jan  1  1970 port
crw-------    1 root     root      108,   0 Dec 27 03:06 ppp
crw-rw-rw-    1 root     root        5,   2 Dec 27 04:32 ptmx
drwxr-xr-x    2 root     root             0 Jan  1  1970 pts
crw-r--r--    1 root     root        1,   8 Jan  1  1970 random
drwxr-xr-x    2 root     root            40 Jan  1  1970 shm
crw-r--r--    1 root     root       10, 254 Dec 27 03:06 switch_ssdk
crw-rw-rw-    1 root     root        5,   0 Jan  1  1970 tty
crw-------    1 root     root        4,  64 Dec 27 03:45 ttyS0
crw-r--r--    1 root     root        1,   9 Jan  1  1970 urandom
crw-r--r--    1 root     root       10, 130 Dec 27 03:06 watchdog
crw-rw-rw-    1 root     root        1,   5 Jan  1  1970 zero
```

## 1. Major number và Minor number - Định danh của device.

Như có thể thấy ở trên, hầu hết các device trong Linux là <span style="color:red">chardev</span>. Ngoài ra, có thể để ý thấy, mỗi device file thường có 2 số được phân tách bởi dấu phẩy đi kèm nhau, ví dụ: với device <span style="color:green">ttyS0</span> hai số này là 4 và 64. Hai số này được gọi là <span style="color:red">Major</span> và <span style="color:red">Minor</span>, chúng được dùng để định danh device trong một rừng cái device của hệ thống. 
Major number (số trước dấu phẩy): Dùng để xác định driver nào sẽ liên kết với thiết bị, ví dụ <span style="color:green">ttyS0</span> sẽ được điều khiển bời driver 4, trong khi các <span style="color:green">mtd</span> device được điều khiển bởi driver 31.
Minor number (số sau dấu phẩy): Cũng ở ví dụ trên, ta có thể thấy 8 <span style="color:green">mtd</span> device có chung một major number, nhưng minor number khác nhau, tức là Minor number được sử dụng để phân biệt các device sử dụng chung một driver.

Trong kernel, có sử dụng một kiểu biến là  dev_t (<span style="color:blue">linux/types.h</span>), kiểu này đượuc dùng để lưu dữ các device numbers(major và minor), nói cách khác, các device trong kernel được phân biệt bằng các biến có kiểu này.
dev_t có kích thước là 32-bit, với 12 bit dùng cho Major number và 20 bit còn lại cho minor number, lưu ý là thứ tự của 2 phần này không được chỉ rõ, nên tốt nhất chúng ta không nên tự xác định major và minor bằng tay từ device number, thay vào đó, kernel source cung cấp cho chúng ta 2 macro để làm việc này là: ```MAJOR(dev_t dev)``` và ```MINOR(dev_t dev)```. Cả hai Macro này đều được định nghĩa trong (<span style="color:blue">linux/kdev_t.h</span>).
Tương tự, nếu như chúng ta đã biết major và minor của một device, chúng ta có thể tìm ra device number của nó bằng Macro: ```MKDEV(int major, int minor)```

## 2. Các cấu trúc dữ liệu quan trọng.
Hầu hết các tác vụ cơ bản của driver gọi đến 3 kernel data structure quan trọng, đó là: file_operations, file và inode.
### 2.1. File Operations - fops
(linux/fs.h)
<span style="color: red">file_operations</span>
file operation structure giúp kết nối các tác vụ của driver với các device number. Mỗi file đang mở (biểu diễn bởi file structure) được liên kết với một tập các function (f_op). Các function này sẽ thực hiện các tác vụ của driver. Nói theo ngôn ngữ OOP thì các file là các Object còn các funtion thực hiện trên file là các method của nó.
Mỗi trường trong file operation structure phải trỏ đến một function được định nghĩa bởi driver, function này thực hiện một tác vụ cụ thể. Nếu một trường nào đó là NULL thì có nghĩa là tác vụ đấy không được hỗ trợ. Mỗi tác vụ này sẽ có các hàm system call tương ứng để có thể gọi đến từ user space.
Sau đây là danh sách các trường của ```struct file_operations```

<div>
<p class="text-uppercase">FPI Warning: It's boring</p>

<p style="background-color: lightblue;"><code>struct module *owner;</code>
	/*
		Đây là trường đầu tiên của fops struct, nó không phải là một tác vụ mà là một con trỏ trỏ đến module sử hữu structure này. 
		Tác dụng: Ngăn chặn việc unload module khi các tác vụ của nó chưa hoàn thành.
		Thông thường nó được khởi tọa bằng giá trị THIS_MODULE (linux/modules.h)
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>loff_t (*llseek) (struct file *, loff_t, int);</code>
	/*
		Được sử dụng để thay đổi vị trí đọc/ghi hiện tại ở trong file. (seek/fseek function in C)
		Return value: vị trí vừa mới chuyển tới, hoạc giá trị âm nếu thao tác lỗi.
		Nếu function pointer là NULL, thì seek call sẽ sửa vị trí hiện tại trong file struct theo cách không xác định được.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);</code>
	/*
		Được sử dụng để lấy dữ liệu từ device. 
		Return value: số lượng bytes đã đọc thành công.
		NULL pointer: read system call sẽ trả về -EINVAL error.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*aio_read) (struct kiocb *, char __user *, size_t, loff_t *);</code>
	/*
		Đọc không đồng bộ - tác vụ đọc có thể không hoàn thành trước khi hàm return.
		NULL pointer thì nó sẽ tự trỏ đến hàm read.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*write) (struct file*, const char __user *, size_t, loff_t *);</code>
	/*
		Hàm này tương tự hàm read.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*aio_write) (struct kiocb *, const char __user *, size_t, loff_t *);</code>
	/*
		Hàm này tương tự hàm aio_read
		Cả hai hàm write đều sử dụng const char => tránh sửa đổi dữ liệu truyền vào.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*readdir) (struct file *, void *, filldir_t);</code>
	/*
		Để đọc các thư mục. tốt nhất là bỏ qua nó
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>unsigned int (*pool)(struct file *, struct poll_table_struct *);</code>
	/*
		Đây là backend của 3 system call: poll, epoll và select
		--Tạm thời ignore, vì chưa tìm hiểu =))
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>unsigned int (*ioctl) (struct inode *, struct file *, unsigned int, usigned long);</code>
	/*
		ioctl system call cung cấp một phương thức để tạo ra các câu lệnh cho các device cụ thể.
		Nếu ioctl không được implement thì sẽ báo lỗi "No such ioctl for device".
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*mmap) (struct file*, struct vm_area_struct *);</code>
	/*
		mmap được sử dụng để mapping device memory tới không gian bộ nhớ của process.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*open) (struct file*, struct file *);</code>
	/*
		- Đây là tác vụ được thực hiện đầu tiên của device file.
		- Có thể có hoặc không.
		- Nếu không có thì return value luôn là true.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*flush) (struct file *);</code>
	/*
		- được gọi khi một process đóng file descriptor (chỉ là bản copy thôi), nó sẽ thực thi và đợi các tác vụ chưa giải quyết xong trên device.
		- Hiếm khi được sử dụng.
		- Nếu NULL thì kernel sẽ bỏ qua request từ user space.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*release) (struct inode *, struct file *);</code>
	/*
		Có open thì phải có release thôi.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*fsync) (struct file *, struct dentry *, int);</code>
	/*
		Flush any pending data.
		NULL -> -EINVAL
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*aio_fsync) (struct kiocb *, int);</code>
	/*
		fsync khÔng đồng bộ
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*fasync) (int, struct file *, int);</code>
	/*
		- ?????????????????????????????
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*lock) (struct file*, int, struct file_lock *);</code>
	/*
		- Được sử dụng để lock file, thường thì không được thực hiện bởi ldd.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*readv) (struct file*, const struct iovec*, unsigned long, loff_t *);</code></p><br/>
<p style="background-color: lightblue;"><code>ssize_t (*write) (struct file*, const struct iovec*, unsigned long, loff_t *);</code>
	/*
		- pending
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*sendfile) (struct file*, loff_t, size_t, read_actor_t, void *);</code>
	/*
		- Được sử dụng bởi sendfile system call.
		- Di chuyển dữ liệu từ một file descriptor tới một file descriptor khác với khối lượng copy nhỏ nhất. (tức là chỉ có gắng gửi những thứ đã bị thay đổi?).
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>ssize_t (*sendpage) (struct file*, struct page *, int, size_t, loff_t *, int);</code>
	/*
		sendpage giống sendfile, nhưng chỉ send từng page chứ không phải là cả file.
		Thông thường không mấy ai implement hàm này.
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>unsigned log (*get_unmmapped_area) (struct file *, unsigned long, unsigned long, unsigned long, unsigned long);</code>
	/*
		- pending
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*check_floags) (int);</code>
	/*
		- CHo phép một module kiểm tra các cờ đã được truyền vào fcntl();
	*/
</p>
<br/>
<p style="background-color: lightblue;"><code>int (*dir_notify) (struct file*, unsigned long);</code>
	/*
		directory change notification.
	*/
</p>

</div>
### 2.2. File Struct - filp
Where is it? => linux/fs.h
File struct đại diện cho một open file (file đang mở). (mọi open file trong hệ thống đều có một struct file tương ứng trong kernel space). Thời gian tồn tại của nó là từ lúc open() call đến lúc close() call, trong suốt thời gian đó, nó được truyển cho mọi function có thao tác trên file.
 Sau đây là những trường quan trọng nhất của cấu trúc này.

<code>mode_t f_mode;</code>
 	/*
 		FMODE_READ|FMODE_WRITE - readable|writeable.
 	*/

<code>loff_t fpos;</code>
 	/*
 		- VỊ trí đọc/ghi hiện tại. 
 		- Driver có thể kiểm tra giá trị của fpos, nhưng không nên thay đổi nó một cách thông thường, thay vào đó, nó được thay đổi bởi read() và write()
 	*/

<code>unsiged int f_flags;</code>
 	/*
 		File flags: O_RDONLY, O_NONBLOCK, O_SYNC.
 		Các flag này được định nghĩa trong linux/fcntl.h
 	*/

<code> struct file_operation *f_op;</code>
 	/*
 		- Các tác vụ gắn với file.
 		- Do kernel không lưu lại f_op cho các lần tham chiếu sau đấy, nên chúng ta có thể thay đổi file_operation gắn với file bất cứ lúc nào và điều này sẽ gây ảnh hưởng ngay lập tức.
 	*/

<code>void *private_data;</code>
 	/*
 		- Trước khi gọi system call open(), đây là một con trỏ NULL.
 		- Có thể dùng hoặc không dùng đều được.
 		- Dùng để lưu giữ các thông tin cố định thông suốt quá trình sử dụng của device.
 		- Cần giải phóng ở hàm release()
 	*/
<code>struct dentry *f_dentry;</code>
 	/*
 		- Directory entry gắn với file.
 	*/

### 2.2. inode struct

- Kernel dùng inode để đại diện cho các file. Lưu ý rằng struct file là đại diện cho descriptor của các file đang mở (fd), do đó với mỗi file trong hệ thống, có thể có nhiều fd nhưng chỉ có duy nhát một inode.
- Chứa các thông tin hữu ích về file:
<p><code>dev_t i_rdev;</code>
	/*
		Chứa device number;
	*/
</p>
<p>
<code>struct cdev *i_cdev;</code>
	/*
		cdev là cấu trúc biểu diễn char devices;
	*/
</p>

## 3. Thử lập trình một Character device driver
<div>
Bây giờ mình sẽ thử tạo một character device driver đơn giản tên là <i>scull</i>, ví dụ này được lấy từ <a href="https://lwn.net/Kernel/LDD3/">Linux Device Driver(ldd3)</a>.

Device driver này hoạt động như một buffer, nó không làm gì khác ngoài việc quản lý các phần bộ nhớ mà bạn có thể đọc hoặc ghi lên đấy. Bộ nhớ được quản lý bằng cách cấu trúc thành một danh sách liên kết, mỗi node là một scull_qset, mỗi scull_qset lưu giữ một vùng dữ liệu và một con trỏ tới qset tiếp theo. Vùng dữ liệu ở đây là một bảng các phần tử (quantums). Trong mỗi bảng này có SCULL_NUM_QUANTUM phần tử, mỗi cái có kích thước SCULL_QUANTUM_SIZE bytes.
Ngoài ra, ở đây còn sử dụng một struct là scull_dev, chứa các thông tin mà device cần đến.
</div>
### 3.1 Khai báo các cấu trúc dữ liệu cần thiết.
#### a, Structure qset
<code>
	struct scull_qset{
		void **data; /data region
		struct scull_qset *next; //next qset
	}
</code>
#### b, Structure scull_dev
<code>
	struct scull_dev{
		struct scull_qset *data; 	//Pointer to the first qset.
		struct quantum;				//Current size of each quamtum.
		int qset; 					//Number of qset?
		unsigned long size;			//Amount of data store here.
		unsigned int access_key;
		struct semaphore sem;		//Semaphore
		struct cdev cdev;			//Cdev struct
	}
</code>
### 3.2 Đăng ký device driver với kernel.
Kernel sử dụng <span style="color:blue">struct cdev</span> để đại diện cho các character device. Để kernel có thể thực thi các
tác vụ có trong driver, chúng ta cần đăng ký struct này với kernel.
<span style="color:blue">struct cdev</span> nằm trong header <span style="color:blue">linux/cdev.h</span>
<div>
	Có hai cách để đăng ký driver với kernel. Đầu tiên, trong trường hợp chỉ muốn đăng ký duy nhất <span style="color:blue">struct cdev</span>
	thì có thể dùng:<br/>
		<code>
			struct cdev *my_cdev = cdev_alloc(); <br/>
			my_cdev->ops = &my_fops; <br/>
		</code>
</div>
<div>
	Trường hợp thứ 2, chúng ta muốn device driver có một cấu trúc dữ liệu riêng để lưu giữ các thông tin của chardev, trong đó
	có chứa cả <span style="color:blue">struct cdev</span>, thì chúng ta sẽ sử dụng cách sau:
		<code>
			void cdev_init(struct cdev *dev, struct file_operations *fops); <br/>
			int cdev_add(struct cdev *dev, dev_t num, unsigned int count); <br/>
		</code>
	Đây là cách mà scull driver sử dụng. <br/>
	Giải thích các tham số:<br/>
	- <span style="color: blue">struct cdev *dev</span> : Đây là cấu trúc cdev đại diện của chardev.
	- <span style="color: blue">struct file_operations *fops</span> : Cấu trúc này chứa các function pointer đến các hàm thực hiện các tác vụ của chardev.
	- Hàm <span style="color: blue">int cdev_add</span> đăng ký chardev với kernel với <span style="color: blue">dev_t num</span> là device number đầu tiên, còn <span style="color: blue">int count</span> là số lượng device number sẽ liên kết với device. Hàm này trả về 0 nếu thành công và ngược lại.
</div>
<div>
	<p>scull driver khởi tạo và thêm cấu trúc cdev của nó vào hệ thống bằng cách sau:</p>
	<code>
		static void scull_setup_dev(struct scull_dev *dev, int index)
		{
			int err, devno = MKDEV(scull_major, scull_minor + index);
			cdev_init(&dev->cdev, &scull_fops);
			dev->cdev.owner = THIS_MODULE;
			dev->cdev.ops = &scull_fops;
			err = cdev_add(&dev->cdev, devno, 1);
			if(err)
				printk(KERN_NOTICE "Error %d adding scull%d",err,index);
		}
	}
	</code>
</div>
### 3.3 Các hàm của cấu trúc file_operations
#### a. open and release
<div>
open(): thực hiện các khởi tạo cơ bản để giúp các tác vụ khác có thể hoạt động sau đó.
Thông thường, hàm open() sẽ thực hiện các nhiệm vụ sau:
- Kiểm tra xem device đã sẵn sàng chưa? Hardware có vấn đề gì không?....
- Khởi tạo device nếu nó được mở lần đầu.
- Cập nhật f_op nếu cần tiếp.
- Cấp phát và gán các thông tin cần thiết vào filp->private_data.

Tuy nhiên, Mục tiêu hàng đầu là xác định xem device nào sẽ được mở (tức là cái file device nào ấy). 
</div>
```
int scull_open(struct inode *inode, struct file *filp)
{
	struct scull_dev *dev; 
	dev = container_of(inode->i_cdev, struct scull_dev, cdev);
	filp->private_data = dev;

	if((filp->f_flags & O_ACCMODE) == O_WRONLY)
	{
		scull_trim(dev);
	}
	return 0;
}
```

release(): Hàm này dùng để phá hoại hết những gì đã làm trong hàm open. Đầu tiên là phải thu deallocate filp->private_data. Poweroff device trong lần dùng cuối. trong scull hàm này không làm gì cả vì không có gì để giải phóng hay power off hết.
Trong kernel, có một counter dùng để đếm xem một <i>file</i> structure có bao nhiêu đối tượng đang sử dụng nó. Khi counter bằng này có giá trị bằng 0 thì đó được xem là lần sử dụng cuối của device và nó sẽ bị poweroff. Ngoài ra counter cũng đảm bảo là mỗi lời gọi đến open() sẽ chỉ có một lời gọi đến release() đi kèm (tránh release 1 file 2 lần).

#### b. read and write

```ssize_t read(struct file *filp, char __user *buff, ssize_t count, loff_t *offp);```
```ssize_t write(struct file *filp, const char __user *buff, ssize_t count, loff_t *offp);```
Hàm này sẽ gửi một buffer có kích thước count bytes tới app space bắt đầu tự vị trí offp của file.
Do buff là user-space pointer nên nó không thể được truy vấn một cách trực tiếp từ kernel code, sau đây là một số hạn chế:
- Phụ thuộc vào arch của hệ thống và configuration của kernel, user-space pointer có thể là không hợp lệ đối với kernel mode. (Do kernel memory là direct mapping, trong khi ở user-space không phải là direct mapping nên cùng một địa chỉ nhưng vị trí sẽ khác nhau).
- User-mem được paged (paging) nên nó không tồn tại lâu dài trong RAM. Việc tham chiếu đến user-space mem một cách trực tiếp sẽ gây ra page fault (không phải lúc nào cũng xảy ra nhưng xác suất cao) kể cả nếu pointer trong kernel-space và user-space có cách mapping giống nhau.
- Về vấn đề bảo mật, việc tham chiếu trực tiếp đến pointer của user-space cũng không tốt vì nó tạo ra risk cao. 

Mặc dù có những hạn chế ở trên, nhưng rõ ràng là chúng ta vẫn cần truy cập đến user-space buffer để hoàn thành việc read(và cả write) của ldd. Kernel cung cấp cho ta các hàm để thực hiện điều này một cách an toàn (thank torvalds). Những hàm này được định nghĩa trong header <span style="color: red">asm/uaccess.h</span>. Những hàm này đã sử dụng ma thuật hắc ám của kẻ mà ai cũng biệt là ai để truyền dữ liệu giữa kernel và user space một cách an toàn và im lặng. Trong phần read(), write() chúng ta cần đến phép thuật sau:
```usigned long copy_to_user(void __user *to, const void *from, usinged long count);```
```usigned long copy_from_user(void __user *to, const void __user *from, usinged long count);```
Lưu ý là do user-space sử dụng cơ chế paging/swapping nên tại thời điểm bất kỳ, có thể page cần dùng để copy/send data không nằm trong bộ nhớ, do đó cần có thời gian để transfer các page này vào mem, điều này đồng nghĩ với việc các hàm read/write phải sleepable ở đây, nên các hàm này sẽ thực hiện một cách concurrently với các hàm khác của driver. 
Hai hàm này không phải là atomic, tức là nó sẽ kiểm tra xem user-space pointer có hợp lệ hay không. Nếu không, việc copy sẽ không được thực hiện, nếu có nó sẽ thực hiện, nhưng giả dụ trong lúc đang copy nó phát hiện ra một địa chỉ không hợp lệ, quá trình copy sẽ bị break và phần data chưa copy sẽ không được xử lý, phần đã copy thì vẫn giữ nguyên. Giá trị trả về của các hàm này đều là lượng data đã copy (bytes).

- Cần update *offp sau khi thực hiện read/write để đảm bảo rằng vị trí hiện tại là đúng.
- Nếu thao tác đọc/ghi không thành công thì giá trị trả về là 1 số ÂM.
b1. read()
Với mỗi giá trị trả về của hàm read(), có một tác động tương ứng lên chương trình (app space) có lời gọi hàm đến nó.
- Nếu giá trị trả về bằng với count thì toàn bộ dữ liệu yêu cầu đã được truyền thành công. Đây là trường hợp tối ưu.
- Nếu giá trị là dương, nhưng nhỏ hơn count, chỉ một phần của dữ liệu đã được truyền thành công. Điều này có thể xảy ra do một số thế lực hắc ám phụ thuộc vào pháp sư sử dụng nó (device). Trường hợp này thường xảy ra khi user-space program gọi đến read().
- Nếu giá trị là 0 thì không có data để truyền đi nữa (chakra cạn kiệt).
- Nếu giá trị trả về là 0, thì tức là nó đã bị phong ấn ở đâu đấy.
- Trường hợp cá biệt, chakra vẫn còn nhưng bị bakugan phong tỏa huyệt đạo, shinobi sẽ rơi vào trang thái block.
Sau đây là source code hàm read của scull
```ssize_t scull_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data;
	struct scull_qset *dptr;
	int quantum = dev->quantum, qset = dev->qset;
	int itemsize = quantum*qset;
	int item, s_pos, q_pos, rest;
	if (*f_pos >= dev->size)
		goto out;								//Nếu f_pos vượt quá kích thước device size tức là end-of-file thì return
	if (*f_pos + count > dev->size)
		count = dev->size -*f_pos;				//truncate buf size thành kích thước còn lại của device size.
	item = (long)*f_pos / itemsize;				//vị trí của q_set cần đọc.
	rest = (long)*f_pos % itemsize;				//vị trí của quantum trong q_set đó
	s_pos = rest/quantum; 						//Đây là index 1 trong **data của qset
	q_pos = rest % quantum;						//Đây là vị trí của byte trong **data;
	dptr = scull_follow(dev, item); 			//traveling linked list.

	if(dptr == NULL || !dptr->data || !dptr->data[s_pos])
		goto out;

	if (count>quantum - q_pos)
		count = quantum - q_pos;
	if(copy_to_user(buf, dptr->data[s_pos]+q_pos,count))
	{
		retval = -EFAULT;
		goto out;
	}

	*f_pos += count;
	retval = count;
	out:
		up(&dev->sem);
		return retval;
}```

b2. write()
giống read, write có thể truyền ít hơn dữ liệu được yêu cầu, sau đây là các giá trị trả về ở user-space calling tương ứng.
- Nếu giá trị trả về bằng count thì toàn bộ các bytes được yêu cầu đã truyền thành công.
- Nếu giá trị trả về là giá trị dương lớn hơn count, thì chỉ một phần chakra được truyền từ cửu vĩ sang naruto. Chương trình (user-space) gần như ngay lập tức cố gắng write phần data còn lại.
- Nếu giá trị trả về là 0 thì tức là không có ghì để write.
- Nếu giá trị trả về là âm thì đã có lỗi.

```python
ssize_t scull_write(struct file *filp, const char __user *buf, size_t count, loff_t *f_pos)
{
	struct scull_dev *dev = filp->private_data;
	struct scull_qset *dptr;
	int quantum = dev->quantum, qset=dev->qset;
	int itemsize = quantum * qset;
	int item, s_pos, q_pos, rest;
	ssize_t retval = -ENOMEM; 

	item = (long)*f_pos/itemsize;
	rest = (long)*f_pos%itemsize;
	s_pos = rest/quantum; q_pos=rest%quantum;

	dptr = scull_follow(dev, item);
	if(dptr == NULL)
		goto out;
	if(!dptr->data)
	{
		dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
		if(!dptr->data)
			goto out;
		memset(dptr->data, 0, q_set * sizeof(char *));
	}

	if(!dptr->data[s_pos]){
		dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL)
		if(!dptr->data[s_pos])
			goto out;
	}

	if(count > quantum - q_pos)
		count = quantum - q_pos;
	if(copy_from_user(dptr->data[s_pos]+q_pos, buf, count))
	{
		retval = -EFAULT;
		goto out;
	}
	*f_pos+=count;
	retval = count;

	if(dev->size < *f_pos)
		dev->size = *f_pos;
	out:
		up(&dev->sem);
		return retval;
}```
