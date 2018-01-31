# BLOCKING I/O

Trong character driver, có implement hai hàm: read() và write(). Thử tưởng tượng, nếu như driver không thể thỏa mãn được request một cách tức thời, thì nó sẽ phản ứng như thế nào? Điều này tức là driver sẽ làm gì nếu như hàm <code>read()</code> được gọi khi dữ liệu chưa sẵn có, nhưng có thể nó sẽ có trong tương lai gần. Hoặc một process cố gắng thực hiện lệnh <code>write()</code> nhưng device chưa sẵn sàng nhận dữ liệu bởi vì buffer đang full. Trong trường hợp này, driver nên block process (user-space) lại, đưa nó vào tình trạng sleep cho đến khi các request có thể được thực thi.<br/>

## 1 Sleeping trong kernel
-Khi một process ở trạng thái sleep, nó sẽ bị remove khỏi scheduler's run queue cho đến khi statuc của nó được thay đổi bởi một sự kiện nào đó. <br/>
-Một process ở sleep status sẽ không được lập lịch trong CPU.<br/><br/>
Để một đoạn code có thể được đưa vào trạng thái sleep thì nó cần thỏa mãn các điều kiện sau đây:<br/>
-Rule 1: Không sleep khi đang trong một atomic context. Tức là driver không được sleep khi đang giữ spinlock, seqlock hoặc RCU lock.<br/>
-Rule 2: Không thể sleep khi đang có các disabled interrupt.<br/>
-Rule 3: Sleep trong semaphore là được phpes, nhưng cần phải cẩn thận. Nếu một đoạn code sleep trong khi nó đang giữ semaphore thì thread đang đợi semaphore đó cũng sẽ bị đưa vào trạng thái sleep, do đó cần đảm bảo rằng bạn không block luôn kthread sẽ đánh thức sleeping process.<br/>
-Rule 4: Khi một sleeping kthread được wake up thì nó không thể biết là nó đã bị loại ra khỏi CPU được bao lâu, và trong thời gian đó, đã xảy ra những thay đổi nào. Do đó, chúng ta không thể tạo ra một giả thuyết nào về trạng thái của hệ thống sau khi wake up kthread, mà phải kiểm tra xem những điều kiện cần thiết có đảm bảo không.<br/>
-Rule 5: Chỉ sleep khi chắc chắn rằng có một ai đó sẽ đánh thức bạn trong một thời điểm nào đó. Ngược lại hệ thống sẽ bị hang. Để làm được điều này thì có một yêu cầu nữa là awaker cần phải tìm được bạn để đánh thức.<br/><br/>

Chúng ta sẽ dùng một structure tên là <code>wait_queue_head_t</code> để thực hiện việc sleep của device driver. một queue head có thể được khởi tạo bằng các cách sau:<br/>
-Initialize statically: <code>DECLARE_WAIT_QUEUE_HEAD(name);</code>
-Initialize dynamicly: <br/>
<pre>
	wait_queue_head_t my_queue;
	init_waitqueue_head(&my_queue);
</pre>
<br/>
## 2.Simple Sleep
-Khi một process sleep, nó rơi vào trạng thái chờ đợi, chờ đợi một điều kiện nào đó sẽ đúng trong tương lại. Như đã đề cập, bất kỳ process nào sleep đều phải kiểm tra để chăc chắn rằng điều kiện nó chờ đợi thực sự được thỏa mãn khi nó tỉnh giấc. <br/>
-Linux cung cấp function family: <code>wait_event</code> để thực hiện sleeping:<br>
<code>wait_event(queue, condition)</code><br/>
<code>wait_event_interruptible(queue, condition)</code><br/>
<code>wait_event_timeout(queue,condition,timeout)</code><br/>
<code>wait_event_interruptible_timeout(queue, condition, timeout)</code><br/><br/>
-Sau khi đã sleep, linux cung cấp các hàm sau để đánh thức sleeping process:<br/>
<code>wake_up(wait_queue_head_t *queue)</code><br/>
<code>wake_up_interruptible(wait_queue_head_t *queue)</code><br/><br/>
Hàm <code>wake_up</code> đánh thức tất cả các process trong hàng đợi, phiên bản interruptible đánh thức các process thực hiện một interruptible sleep.<br/>
Lưu ý: interruptible sleep tức là các sleep có thể bị interrupt bởi một tác nhân bên ngoài, thông thường đây là cái chúng ta cần dùng.<br/><br/>

## 3.Blocking and Nonblocking Operations
Phần này sẽ nói về việc xác định xem khi nào chúng ta sẽ đưa process vào trạng thái sleep?<br/>
-Một số operation trong UNIX yêu cầu rằng không được block nó, kể cả nếu như nó không thể thực hiện một cách hoàn toàn. Ngoài ra cũng có một số thời điểm mà process thông tin cho bạn rằng nó không muốn bị block, kể cả nó có thực hiện I/O hay không. Những nonblocking I/O rõ rằng này được thông tin bởi cờ O_NONBLOCK trong <code>filp->f_flags</code>.<br/><br/>
Ở trường hợp blocking operation, đây là mặc định, các thao tác sau đây nên được impelement:<br/>
-Nếu một process gọi hàm <code>read</code> nhưng dữ liệu chưa khả dụng, thì process block. Process được đánh thức ngay khi dữ liệu đến, và dữ liệu được trả về cho người gọi hàm, kể cả nếu lượng dữ liệu trả về ít hơn lượng dữ liệu được yêu cầu (trong argument <code>count</code>).<br/>
-Nếu một process gọi hàm <code>write</code> và buffer đầy, process phải block, và nó phải nawmgf ở một wait queue khác so với wait queue đang được sử dụng cho việc reading. Khi một số dữ liệu được ghi vào hardware device, và buffer bắt đầu có không gian trống, process sẽ được đánh thức và <code>write</code> được gọi thành công, mặc dù dữ liệu có thể chỉ được ghi một nửa so với lượng dữ liệu được yêu cầu.<br/><br/>
Trường hợp O_NONBLOCK flag được set, nonblocking operations nó sẽ return ngay lập tức, cho phép application lấy dữ liệu. Application phải cẩn thận khi sử dụng các <code>stdio</code> function khi đang sử lý các nonblocking files. Cần phải check <code>errno</code>.<br/>
Một cách tự nhiên, O_NONBLOCK rất có ý nghĩa đối với <code>open</code>. Điều này diễn ra khi lời gọi có thể bị block một thời gian dài, ví dụ, khi mở một FIFO mà tạm thời nó chưa có writer, hoặc khi truy cập một disk file với pending lock. Thông thường, việc mở một device có thể thành công hoặc thất bại, mà không cần phải đợi các event bên ngoài. Tuy nhiên, đôi khi việc mở một device yêu cần một thời gian lâu hơn, và chúng ta có thể chọn sử dụng O_NONBLOCK trong hàm <code>open</code> bằng cách trả về lỗi -EAGAIN ngay lập tức nếu như cờ block được set, sau khi bắt đầu tiến trình khởi tạo device.<br/>

## 4.Example
Sau đây mình sẽ trình bày một ví dụ sleep đơn giản.
Đầu tiên tạo file source code <code>oni_sleep.c</code>. Tiếp theo là include các header cần thiết: <br/>
<pre>
#include <linux/module.h>
#include <linux/uaccess.h>
</pre>

Tiếp theo chúng ta define các hằng số cần thiết cho việc xác định device number:<br/>
<pre>
#define MINOR_FIRST 0
#define MINOR_COUNT 1
</pre>

Khai báo các biến cần thiết cho device driver <br/>
<pre>
static struct cdev oni_sleep_cdev;
static struct class *oni_sleep_class;
static struct device *oni_sleep_device;
static dev_t device_number;
static int sleep_flag=0;
</pre>

Để thực hiện việc sleep, cần phải khai báo một head queue, ở đây mình khai báo head queue bằng cách dùng DECLARE (Statically)
<pre>
DECLARE_WAIT_QUEUE_HEAD(oni_wait_queue);
</pre>

Ở đây ngoài các cấu trúc quen thuộc với một device driver, mình có khai báo thêm biến sleep_flag, biến này sẽ được dùng để làm điểu kiện wake up cho process đã bị sleep trước đó (sleeping process sẽ được đánh thức khi sleep_flag=1)<br/>
Tiếp đến là định nghĩa 2 function của file_operations: open() và release()
<pre>
int sleep_open(struct inode *node, struct file *filp)
{
	//Do nothing at the moment
	return 0;
}

int sleep_release(struct inode *node, struct file *filp)
{
	//Do nothing at the moment
	return 0;
}
</pre>
Tạm thời 2 function này sẽ không làm gì cả <br/>
Kế tiếp là function read. Ở đây hàm read sẽ không thực hiện đọc dữ liệu hay làm gì khác cả. Nó chỉ ghi ra các log để trace quá trình sleep và wakeup mà thôi. <br/>
<pre>
ssize_t sleep_read(struct file *filp, char __user *buf, size_t count, loff_t *pos)
{
	printk(KERN_INFO "[READ] Enter read function\n");
	printk(KERN_WARNING "[READ] About to sleep\n");
	wait_event_interruptible(oni_wait_queue, sleep_flag == 1);
	printk(KERN_INFO "[READ] read woken up\n");
	return 0;
}
</pre>
Ở hàm read, sau khi in ra 2 log line (dmesg), mình đã block hàm <code>read</code>, và chờ cho điều kiện <code>sleep_flag==1</code> thì sẽ đánh thức nó dậy. Mình sử dụng head queue đã khai báo ở đầu, đưa process vào wait_interruptible, tức là người dùng có thể Ctrl+C để thoát user-app khi nó đang đợi <code>read</code> trả về.<br/>
Bây giờ, mình sẽ wake up hàm <code>read</code> từ hàm <code>write</code>, hiện tại hàm write cũng chỉ được dùng để đánh thức hàm read từ hàng đợi mà không có ghi dữ liệu gì sất. Mình sẽ dùng <code>wake_up</code> để đánh thức tất cả các entry trung head queue dậy.
<pre>
ssize_t sleep_write(struct file *filp, const char __user *buf, size_t count, loff_t *pos)
{
	printk(KERN_INFO "[WRITE] Enter write function\n");
	printk(KERN_INFO "[WRITE] Wake read up ZZZZZZ\n");
	wake_up(&oni_wait_queue);
	printk(KERN_INFO "[WRITE] Exit write\n");
}
</pre>

Chúng ta đã có đầy đủ các file operation cần thiết, bây giờ chúng ta có thể khai báo struct file_operations cho oni_sleep
<pre>
static struct file_operations oni_fops={
	.open = sleep_open,
	.release = sleep_release,
	.read = sleep_read,
	.write = sleep_write,
	.owner = THIS_MODULE	
};
</pre>

Bây giờ chúng ta đã có đủ các thành phần để bắt đầu khởi tạo module, hãy bắt tay vào việc:
<pre>
static int __init oni_sleep_init(void)
{
	int ret;
	ret = alloc_chrdev_region(&device_number, MINOR_FIRST, MINOR_COUNT, "oni_sleep");
	if(ret != 0)
	{
		printk(KERN_WARNING "Cannot alloc a region of device number\n");
		return ret;
	}

	cdev_init(&oni_sleep_cdev, &oni_fops);
	ret = cdev_add(&oni_sleep_cdev, device_number, MINOR_COUNT);
	if ( ret != 0 )
	{
		printk(KERN_WARNING "Cannot register module with kernel\n");
		return ret;	
	}
	printk(KERN_INFO "Regitered module with kernel\n");

	oni_sleep_class = class_create(THIS_MODULE, "oni_sleep");
	if (IS_ERR(oni_sleep_class))
	{
		unregister_chrdev_region(device_number, MINOR_COUNT);
		printk(KERN_WARNING "Cannot create device class\n");
		return PTR_ERR(oni_sleep_class);	
	}
	printk(KERN_INFO "Created device class\n");

	oni_sleep_device = device_create(oni_sleep_class, NULL, device_number, NULL, "oni_sleep");
	if (IS_ERR(oni_sleep_device))
	{
		class_destroy(oni_sleep_class);
		cdev_del(&oni_sleep_cdev);
		unregister_chrdev_region(device_number, MINOR_COUNT);
		printk(KERN_WARNING "Cannot create device file\n");
		return PTR_ERR(oni_sleep_device);	
	}	
	printk(KERN_INFO "Created device file\n");
	return 0;
}
</pre>

Tiếp theo tất nhiên là hàm exit 
<pre>
static void __exit oni_sleep_exit(void)
{
	device_destroy(oni_sleep_class, device_number);
	class_destroy(oni_sleep_class);
	cdev_del(&oni_sleep_cdev);
	unregister_chrdev_region(device_number, MINOR_COUNT);
	printk(KERN_INFO "Say goodbyte to your hand!\n");
}
</pre>

Những thứ râu ria khác, nhưng rất cần thiết
<pre>
MODULE_LICENSE("GPL");            ///< The license type -- this affects available functionality
MODULE_AUTHOR("Oni Ranger");    ///< The author -- visible when you use modinfo
MODULE_DESCRIPTION("A simple Linux char driver for explain sleeping");  ///< The description -- see modinfo
MODULE_VERSION("0.1");            ///< A version number to inform users
module_init(oni_sleep_init);
module_exit(oni_sleep_exit);
</pre>

Tiếp theo chúng ta cần tạo ra một user-app để tương tác với device driver đã tạo tên là test.c
```
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>
#include<fcntl.h>
#include<string.h>
#include<unistd.h>

#define BUFFER_SIZE 256;

int read_device(int fd)
{
	char fakeRecv[BUFFER_SIZE];
	printf("Press anykey to block your read function\n");
	getchar();
	read(fd, fakeRecv, strlen(fakeRecv));
	printf("Woken up\n");
}

int write_device(int fd)
{
	char fakeInput[]="hello";
	printf("Press anykey to wake up your read function\n");
	getchar();
	write(fd, fakeInput, strlen(fakeInput));
}

int main(int argc, char* argv[])
{
	int fd;
	fd = open("/dev/oni_sleep",O_RDWR);
	if(fd<0)
	{
		perror("Failed to open the device ....");
		return errno;
	}

	if(argc == 2)
	{
		//nếu có tham số dòng lệnh thì nghĩa là write to device
		write_device(fd);
	}else
	{
		read_device(fd);
	}
}
```

Tiếp theo cần một Makefile để compile những source code dã mổ cò
<pre>
obj-m+=oni_sleep.o
all:
 make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
 $(CC) test.c -o test
clean:
 make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
 rm test
</pre>

<br/>Bây giờ hãy thử xem nó hoạt động như thế nào.<br/>
Sau khi đã compile, chúng ta nhận được file oni_sleep.ko và test. Việc tiếp theo là insert oni_sleep: <code>sudo insmod oni_sleep.ko</code><br/>
Mở một tab mới với thực hiện việc đọc từ device với command: <code>sudo ./test</code><br/>
Ở tab này bạn sẽ nhìn thấy output như sau:
<pre>
Press anykey to block your read function
....
</pre>
Sau khi bạn gõ Enter, tab này sẽ bị block tại đây thay vì return. Mở một tab khác với thực hiện việc ghi vào device với command: <code>sudo ./test write</code>. Như đã đề cập ở trên, hàm write sẽ unblock hàm read.<br/>
Bây giờ quay lại tab read, gõ Enter xem process còn bị block hay không? Wth, hàm read vẫn bị block @@. Tất nhiên là thế rồi, vì mặc dù mình đã gọi hàm <code>wake_up</code> để đánh thức hàm read, nhưng điều kiện <code>sleep_flag==1</code> vẫn chưa được thảo mãn nên nó vẫn tiếp tục bị block.<br/>
Bây giờ, mình sẽ set cờ sleep_flag bằng 1 trước khi gọi wake_up, thêm dòng code vào hàm <code>sleep_write</code> như sau:<br/>
<pre>
......
sleep_flag = 1;
wake_up(&oni_wait_queue);
.............
</pre>
Bây giờ compile lại, sau đấy insmod và thực hiện test như cũ. Bạn sẽ thấy tab read được unblock sau khi thực hiện <code>sudo ./test write</code>.<br/>
Bây giờ kiểm tra dmesg:<br/>
<pre>
[READ] Enter read function
[READ] About to sleep
[WRITE] Enter write function
[WRITE] Wake read up ZZZZZZ
[WRITE] Exit write
[READ] read woken up
</pre>