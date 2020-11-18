---
layout: post

category: Linux device driver

comments: true
---
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
 ```struct file_operations``` có rất nhiều trường khác nhau nhưng về cơ bản chỉ cần quan  tâm đến một số hàm và thuộc tính thường dùng bao sau: 

<p style="background-color: lightblue;"><code>struct module *owner;</code>
	/*
		Đây là trường đầu tiên của fops struct, nó không phải là một tác vụ mà là một con trỏ trỏ đến module sử hữu structure này. 
		Tác dụng: Ngăn chặn việc unload module khi các tác vụ của nó chưa hoàn thành.
		Thông thường nó được khởi tọa bằng giá trị THIS_MODULE (linux/modules.h)
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
<p style="background-color: lightblue;"><code>int (*release) (struct inode *, struct file *);</code>
	/*
		Có open thì phải có release thôi.
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
<p style="background-color: lightblue;"><code>ssize_t (*write) (struct file*, const char __user *, size_t, loff_t *);</code>
	/*
		Ngược lại với hàm read.
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

## 3. Thử lập trình một Character device driver (Incomplete)

Bây giờ mình sẽ thử tạo một character device driver đơn giản tên là <code>oni_chardev</code>, driver này không có giao tiếp gì với phần cứng cả, chỉ là một ví dụ để hiểu hơn về cách viết character device driver thôi. Trong device này sẽ lưu lại một string được nhập vào bởi user-app, và in ra nếu user-app yêu cầu.

### 3.1 Khai báo các cấu trúc dữ liệu cần thiết.
Đầu tiên chúng ta khai báo các hằng số cần thiết cho việc đăng ký device number:<br/>
{% highlight c %}
#define MINOR_FIRST 0
#define MINOR_COUNT 1
#define DEV_NAME "oni_chrdev"
#define BUFFER_SIZE 256
{% endhighlight %}
Device của chúng ta sẽ alloc minor bắt đầu tư 0, với tối đa là 1 minor.<br/>
Tiếp theo là các cấu trúc mà mọi cdd đều có, về cơ bản chúng ta sẽ khai báo như sau.:
{% highlight c %}
static struct cdev oni_cdev;
static struct class *oni_class;
static struct class *oni_device;
static size_t size_of_msg=0;
{% endhighlight %}
Một biến kiểu <code>dev_t</code> để lưu giữ device number mà device được cấp phát.
{% highlight c %}
static dev_t oni_device_number;
{% endhighlight %}

Tiếp theo là cho khai báo các signature của các function được sử dụng bởi file_operations:
{% highlight c %}
	static int oni_open(struct inode *, struct file *);
	static int oni_release(struct inode *, struct file *);
	static ssize_t oni_write(struct file *, const char __user *, size_t count, loff_t *pos);
	static ssize_t oni_read(struct file *, char __user *, size_t count, loff_t *pos);
{% endhighlight %}

Sau khi đã khai báo các signature thì chúng ta sẽ định nghĩa file operation được sử dụng bởi device.
{% highlight c %}
struct file_operations oni_fops
{
	.owner = THIS_MODULE,
	.open = oni_open,
	.release = oni_release,
	.write = oni_write,
	.read = oni_read
};
{% endhighlight %}

Cuối cùng là biến để lưu giữ chuỗi ký tự:
{% highlight c %}
char msg[BUFFER_SIZE];
{% endhighlight %}

### 3.2 Đăng ký device driver với kernel.
Việc đầu tiên khi một device driver được insert vào kernel là kernel sẽ gọi đến hàm init của nó. Hàm init sẽ thực hiện việc đăng ký device number, khởi tạo và đăng ký cấu trúc cdev với kernel, ngoài ra nó cũng có thể đăng ký class và device file cho device. Nếu một device không có device file thì user-app không giao tiếp đọc ghi với nó được(đoán thế), tuy nhiên, linux không yêu cầu chúng ta tạo device file khi init module, thay vào đó, chúng ta có thể tạo ra device file sau bằng command <code>mknod</code>. Trong ví dụ này, mình sẽ tạo luôn device file trong hàm init.<br/>
Đầu tiên chúng ta cần có một device number cho device của chúng ta. Ở đây, có thể dùng macro MKDEV() nếu như chúng ta đã xác định sẵn một major number cho device, sao cho nó không trùng với major number của các device khác trong hệ thống, mặc nhiên là cách này chỉ dùng được khi device driver của chúng ta chỉ dùng cho một hệ thống cá nhân của riêng mình. Trong các hệ thống public, có nhiều người sử dụng thì chúng ta không thể biết được liệu người dùng có thêm vào hệ thống một device nào khác có major number giống của chúng ta hay không. Do đó chúng ta sẽ sử dụng phương pháp cấp phát động cho device number, phương pháp này, kernel sẽ cung cấp một major number chưa có ai sử dụng cho device của chúng ta. (thật ra là của tui, nhưng mà viết chúng ta cho nó có vần thôi).
{% highlight c %}
int ret; 
ret = alloc_chrdev_region(&oni_device_number, MINOR_FIRST, MINOR_COUNT,DEV_NAME);
if( ret != 0 )
{
	printk(KERN_WARNING "Cannot allocate a device number");
	return ret;
}
{% endhighlight %}

Trên đây, chúng ta đã đăng ký một device number có major động và minor number từ 0 đến 0. Biến ret sẽ dùng để lưu giá trị trả về của hàm alloc, nếu ret âm thì tức là có lỗi, lúc này chúng ta sẽ return ngay tắp lự.<br/>

Khi đã có được device number, chúng ta sẽ khởi tạo cấu trúc cdev với hàm <code>cdev_init</code>
<code>cdev_init(&oni_dev, &oni_fops);</code>
Với dòng code này, chúng ta đã khởi tạo cấu trúc oni_dev và ghi nhớ oni_fops, sẵn sàng cho việc sử dụng sau này.<br/>
Tiếp theo là thông báo với kernel về sự hiện diện của chúng ta.<br/>
{% highlight c %}
ret = cdev_add(&oni_dev, oni_device_number, MINOR_COUNT);
if( ret != 0 )
{
	unregister_chrdev_region(oni_device_number, MINOR_COUNT);
	printk(KERN_WARNING "Cannot add device to kernel");
	return ret;
}
{% endhighlight %}
Dòng này dùng để thêm device được biểu diễn bởi biến <code>oni_dev</code> (chính là device này đây) vào kernel, cũng gần như ngay lập tức, make device live. Nếu như lời gọi hàm thực hiện không thành công thì chúng ta kết thúc quá trình khởi tạo device, đồng thời giải phóng device number đang nắm giữ.<br/>
Thật ra, chỉ cần như này là device driver đã có thể được insert vào hệ thống với insmod rồi, tuy nhiên chúng ta sẽ tạo class và device file cho nó trong hàm init này luôn.
{% highlight c %}
oni_class = class_create(THIS_MODULE, DEV_NAME);
if (IS_ERR(oni_class))
{
	cdev_del(&oni_cdev);
	unregister_chrdev_region(oni_device_number, MINOR_COUNT);
	printk(KERN_WARNING "Cannot create class");
	return PTR_ERR(oni_class);
}
{% endhighlight %}
Hàm <code>class_create</code> trả về một con trỏ <code>struct class</code>. Vậy class là gì? Cái này không phải class (lớp) trong java hay C++. Cái này tạm gọi là class device.<br>
Các device trong kernel được chia thành nhiều class. Các device trong cùng 1 class thường có chung một chức năng chính. Bạn có thể xem các class hiện có ở dir: /sys/class

Đối với char device, chúng ta có thể tạo device file bằng cách sau:
{% highlight c %}
oni_device = device_create(oni_class, NULL, oni_device_number, NULL, DEV_NAME);
if (IS_ERR(oni_device))
{
	class_destroy(oni_class);
	cdev_del(&oni_cdev);
	unregister_chrdev_region(oni_device_number, MINOR_COUNT);
	printk(KERN_WARNING "Cannot create device file");
	return PTR_ERR(oni_device);
}
{% endhighlight %}
Bằng đoạn code này, kernel sẽ tạo ra file /dev/oni_chrdev, và các user-app có thể giao tiếp với device thông qua file này.
Đến đây chúng ta hoàn thành hàm init rồi hí hí.
{% highlight c %}
	pr_info( "Initialized device driver");
	return 0;
{% endhighlight %}

Vì hàm exit hiện tại không có nhiều việc để làm nên sẽ nói luôn ở đây:
{% highlight c %}
void __exit oni_exit(void)
{
	device_destroy(oni_class, oni_device_number);
	class_destroy(oni_class);
	cdev_del(&oni_cdev);
	unregister_chrdev_region(oni_device_number, MINOR_COUNT);
}
{% endhighlight %}
### 3.3 Các hàm của cấu trúc file_operations
#### a. open and release
open(): thực hiện các khởi tạo cơ bản để giúp các tác vụ khác có thể hoạt động sau đó.
Thông thường, hàm open() sẽ thực hiện các nhiệm vụ sau:
- Kiểm tra xem device đã sẵn sàng chưa? Hardware có vấn đề gì không?....
- Khởi tạo device nếu nó được mở lần đầu.
- Cập nhật f_op nếu cần tiếp.
- Cấp phát và gán các thông tin cần thiết vào filp->private_data.

Tuy nhiên, Mục tiêu hàng đầu là xác định xem device nào sẽ được mở (tức là cái file device nào ấy). <br/>
Hiện tại chúng ta chưa cần đến hàm này, nên chỉ cần định nghĩa 1 hàm thân rỗng là được.

release(): Hàm này dùng để phá hoại hết những gì đã làm trong hàm open. Đầu tiên là phải thu deallocate filp->private_data. Poweroff device trong lần dùng cuối. trong scull hàm này không làm gì cả vì không có gì để giải phóng hay power off hết.
Trong kernel, có một counter dùng để đếm xem một <i>file</i> structure có bao nhiêu đối tượng đang sử dụng nó. Khi counter bằng này có giá trị bằng 0 thì đó được xem là lần sử dụng cuối của device và nó sẽ bị poweroff. Ngoài ra counter cũng đảm bảo là mỗi lời gọi đến open() sẽ chỉ có một lời gọi đến release() đi kèm (tránh release 1 file 2 lần).<br/>
Hiện tại chúng ta chưa cần đến hàm này, nên chỉ cần định nghĩa 1 hàm thân rỗng là được.

#### b. read and write

```ssize_t read(struct file *filp, char __user *buff, ssize_t count, loff_t *offp);```
```ssize_t write(struct file *filp, const char __user *buff, ssize_t count, loff_t *offp);```
Hàm này sẽ gửi một buffer có kích thước count bytes tới app space bắt đầu tự vị trí offp của file.
Do buff là user-space pointer nên nó không thể được truy vấn một cách trực tiếp từ kernel code, sau đây là một số hạn chế:
- Phụ thuộc vào arch của hệ thống và configuration của kernel, user-space pointer có thể là không hợp lệ đối với kernel mode. (Do kernel memory là direct mapping, trong khi ở user-space không phải là direct mapping nên cùng một địa chỉ nhưng vị trí sẽ khác nhau).
- User-mem được paged (paging) nên nó không tồn tại lâu dài trong RAM. Việc tham chiếu đến user-space mem một cách trực tiếp sẽ gây ra page fault (không phải lúc nào cũng xảy ra nhưng xác suất cao) kể cả nếu pointer trong kernel-space và user-space có cách mapping giống nhau.
- Về vấn đề bảo mật, việc tham chiếu trực tiếp đến pointer của user-space cũng không tốt vì nó tạo ra risk cao. 

Mặc dù có những hạn chế ở trên, nhưng rõ ràng là chúng ta vẫn cần truy cập đến user-space buffer để hoàn thành việc read(và cả write) của ldd. Kernel cung cấp cho ta các hàm để thực hiện điều này một cách an toàn (thank torvalds). Những hàm này được định nghĩa trong header <span style="color: red">linux/uaccess.h</span>. Những hàm này đã sử dụng ma thuật hắc ám của kẻ mà ai cũng biệt là ai để truyền dữ liệu giữa kernel và user space một cách an toàn và im lặng. Trong phần read(), write() chúng ta cần đến phép thuật sau:
```usigned long copy_to_user(void __user *to, const void *from, usinged long count);```
```usigned long copy_from_user(void __user *to, const void __user *from, usinged long count);```
Lưu ý là do user-space sử dụng cơ chế paging/swapping nên tại thời điểm bất kỳ, có thể page cần dùng để copy/send data không nằm trong bộ nhớ, do đó cần có thời gian để transfer các page này vào mem, điều này đồng nghĩ với việc các hàm read/write phải sleepable ở đây, nên các hàm này sẽ thực hiện một cách concurrently với các hàm khác của driver. <br/>
Hai hàm này không phải là atomic, tức là nó sẽ kiểm tra xem user-space pointer có hợp lệ hay không. Nếu không, việc copy sẽ không được thực hiện, nếu có nó sẽ thực hiện, nhưng giả dụ trong lúc đang copy nó phát hiện ra một địa chỉ không hợp lệ, quá trình copy sẽ bị break và phần data chưa copy sẽ không được xử lý, phần đã copy thì vẫn giữ nguyên. Giá trị trả về của các hàm này đều là lượng data đã copy (bytes). [Atomic nghĩ là chỉ có 2 trường hợp: chạy hết thành công, trường hợp 2 là chạy thất bại ở một bước nào đấy thì toàn bộ sẽ bị roll back, giống trong SQL].<br/>

- Cần update *offp sau khi thực hiện read/write để đảm bảo rằng vị trí hiện tại là đúng.
- Nếu thao tác đọc/ghi không thành công thì giá trị trả về là 1 số ÂM.
b1. read()
Với mỗi giá trị trả về của hàm read(), có một tác động tương ứng lên chương trình (app space) có lời gọi hàm đến nó.
- Nếu giá trị trả về bằng với count thì toàn bộ dữ liệu yêu cầu đã được truyền thành công. Đây là trường hợp tối ưu.
- Nếu giá trị là dương, nhưng nhỏ hơn count, chỉ một phần của dữ liệu đã được truyền thành công. Điều này có thể xảy ra do một số thế lực hắc ám phụ thuộc vào pháp sư sử dụng nó (device). Trường hợp này thường xảy ra khi user-space program gọi đến read().
- Nếu giá trị là 0 thì không có data để truyền đi nữa (chakra cạn kiệt).
- Nếu giá trị trả về là 0, thì tức là nó đã bị phong ấn ở đâu đấy.
- Trường hợp cá biệt, chakra vẫn còn nhưng bị bakugan phong tỏa huyệt đạo, shinobi sẽ rơi vào trang thái block.
Mặc dù ở trên có đề cập việc thay đổi file offset, tuy nhiên ví dụ của chúng ta mong muốn là đọc ghi từ đầu file, nên không cần phải update nó làm gì cả, (cả read và write).

{% highlight c %}
static ssize_t oni_read(struct file *filp, char __user *buffer, size_t count, loff_t *offset)
{
	int err_count = 0;
	err_count = copy_to_user(buffer, msg, size_of_msg);
	if( err_count == 0 )
	{
		pr_info( "Oni Chrdev: Sent %lu chars to the user\n", size_of_msg);
		size_of_msg = 0;
		return 0;
	}else
	{
		pr_info( "Oni Chrdev: Failed to send %d chars to the user\n", err_count);
		return -EFAULT;
	}
}
{% endhighlight %}
b2. write()
giống read, write có thể truyền ít hơn dữ liệu được yêu cầu, sau đây là các giá trị trả về ở user-space calling tương ứng.
- Nếu giá trị trả về bằng count thì toàn bộ các bytes được yêu cầu đã truyền thành công.
- Nếu giá trị trả về là giá trị dương lớn hơn count, thì chỉ một phần chakra được truyền từ cửu vĩ sang naruto. Chương trình (user-space) gần như ngay lập tức cố gắng write phần data còn lại.
- Nếu giá trị trả về là 0 thì tức là không có ghì để write.
- Nếu giá trị trả về là âm thì đã có lỗi.
{% highlight c %}
static ssize_t oni_write(struct file *filp, const char __user *buffer, size_t count, loff_t *offset)
{
	if(copy_from_user(msg, buffer, count))
	{
		return -EACCES;
	}

	size_of_msg = strlen(msg);
	pr_info( "Oni Chrdev: receive %zu charaters for the user: %s\n",count,msg);
	return count;
}
{% endhighlight %}

File source hoàn chỉnh sẽ như sau:

{% highlight c %}
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/err.h>
#include <linux/uaccess.h>

#define MINOR_FIRST 0
#define MINOR_COUNT 1
#define DEV_NAME "oni_chrdev"
#define BUFFER_SIZE 256

static struct cdev oni_cdev;
static struct class *oni_class;
static struct device *oni_device;
static size_t size_of_msg=0;
char msg[BUFFER_SIZE]={0};

static dev_t oni_device_number;

static int oni_open(struct inode *, struct file *);
static int oni_release(struct inode *, struct file *);
static ssize_t oni_write(struct file *, const char __user *, size_t count, loff_t *pos);
static ssize_t oni_read(struct file *, char __user *, size_t count, loff_t *pos);

struct file_operations oni_fops=
{
	.owner = THIS_MODULE,
	.open = oni_open,
	.release = oni_release,
	.write = oni_write,
	.read = oni_read
};

static ssize_t oni_read(struct file *filp, char __user *buffer, size_t count, loff_t *offset)
{
	if ((*offset + count) > BUFFER_SIZE)	
		count = BUFFER_SIZE - *offset;

	if (copy_to_user(buffer, msg, count))
	{
		pr_info("Oni Chrdev: Failed to send %d chars to the user\n", err_count);
		return -EFAULT;
	}

	*offset += count;
	pr_info("Oni Chrdev: Number of bytes successfully read = %zu\n", count);
	return count;
}

static ssize_t oni_write(struct file *filp, const char __user *buffer, size_t count, loff_t *offset)
{
	if ((*offset + count) > BUFFER_SIZE)
		count = BUFFER_SIZE - *offset;

	if (!count)
		return -ENOMEM;
	if(copy_from_user(msg, buffer, count))
	{
		return -EFAULT;
	}

	*offset ++ count;
	pr_info( "Oni Chrdev: receive %zu charaters for the user %s\n",count,msg);
	return count;
}

static int oni_open(struct inode *node, struct file *filp)
{
	return 0;
}

static int oni_release(struct inode *node, struct file *filp)
{
	return 0;
}

void __exit oni_exit(void)
{
	device_destroy(oni_class, oni_device_number);
	class_destroy(oni_class);
	cdev_del(&oni_cdev);
	unregister_chrdev_region(oni_device_number, MINOR_COUNT);
}

int __init oni_init(void)
{
	int ret; 
	ret = alloc_chrdev_region(&oni_device_number, MINOR_FIRST, MINOR_COUNT,DEV_NAME);
	if( ret != 0 )
	{
		printk(KERN_WARNING "Cannot allocate a device number");
		return ret;
	}
	cdev_init(&oni_cdev, &oni_fops);	
	ret = cdev_add(&oni_cdev, oni_device_number, MINOR_COUNT);
	if( ret != 0 )
	{
		unregister_chrdev_region(oni_device_number, MINOR_COUNT);
		printk(KERN_WARNING "Cannot add device to kernel");
		return ret;
	}
	
	oni_class = class_create(THIS_MODULE, DEV_NAME);
	if (IS_ERR(oni_class))
	{
		cdev_del(&oni_cdev);
		unregister_chrdev_region(oni_device_number, MINOR_COUNT);
		printk(KERN_WARNING "Cannot create class");
		return PTR_ERR(oni_class);
	}
	
	oni_device = device_create(oni_class, NULL, oni_device_number, NULL, DEV_NAME);
	if (IS_ERR(oni_device))
	{
		class_destroy(oni_class);
		cdev_del(&oni_cdev);
		unregister_chrdev_region(oni_device_number, MINOR_COUNT);
		printk(KERN_WARNING "Cannot create device file");
		return PTR_ERR(oni_device);
	}
	return 0;
}

MODULE_LICENSE("GPL");            ///< The license type -- this affects available functionality
MODULE_AUTHOR("Oni Ranger");    ///< The author -- visible when you use modinfo
MODULE_DESCRIPTION("A simple Linux char driver for explain sleeping");  ///< The description -- see modinfo
MODULE_VERSION("0.1");            ///< A version number to inform users
module_init(oni_init);
module_exit(oni_exit);

{% endhighlight %}

Chúng ta cần một file Makefile để compile module vừa viết, make file tương đối đơn giản:
{% highlight make %}
obj-m+=oni_chrdev.o
all:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
{% endhighlight %}

### 3.4 Vọc vạch cái device driver vừa viết.
Bây giờ insert module vào kernel: <code>sudo insmod oni_chardev.ko</code><br/>
Tiếp theo chúng ta ghi một đoạn ký tự vào device: <code>sudo echo "testing">/dev/oni_chardev</code><br/>
Kiểm tra <code>dmesg</code> xem có gì xảy ra<br/>
Đọc dữ liệu từ device: <code>cat /dev/oni_chardev</code>