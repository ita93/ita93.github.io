# Character Device Driver.

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
```file_operations```
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
 		Các flag này được định nghĩa trong <linux/fcntl.h>
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
Thời gian học đại học ở VNU, về cơ bản các môn lập trình cơ bản đều có một bài tập dạng quản lý sinh viên, hoặc tương tự.
Bây giờ mình sẽ viết một chương trình quản lý sinh viên như thế bằng char dev. Thông tin của mỗi sinh viên sẽ bao gồm: họ tên và ngày sinh. Các thông tin này được lưu giữ trong một struct như sau

</div>
 <pre><code>
	typedef struct{
		char* firstName;
		char* lastName;
		int year;
		int day;
		int month;	
	}Student;
	</pre>
 <pre><code>

