---
layout: post

category: Linux device driver

comments: true
---
Blocking IO dịch một cách đơn giản nghĩa là thực thi đơn luồng, tức là tác vụ chỉ có thể được thực thi sau khi một hay một số tác vụ nhất định nào đấy đã được hoàn thành. Ví dụ như chương trình Word chỉ có thể đọc một file văn bản <u><b>sau khi</b></u> nó đã mở được file đó.

Trong character driver, có implement hai hàm: read() và write(). Thử tưởng tượng, nếu như driver không thể thỏa mãn được yêu cầu một cách tức thời, chẳng hạn như  driver sẽ làm gì nếu như hàm <code>read()</code> được gọi khi dữ liệu chưa sẵn có, nhưng có thể nó sẽ có trong tương lai gần. Hoặc một process cố gắng thực hiện lệnh <code>write()</code> nhưng device chưa sẵn sàng nhận dữ liệu bởi vì buffer đang full. Trong trường hợp này, driver nên block process (user-space) lại, đưa nó vào tình trạng sleep cho đến khi các yêu cầu đó có thể được thực thi.<br/>

## 1 Sleeping trong kernel
-Khi một process ở trạng thái <i>sleep</i>, nó sẽ bị loại bỏ khỏi Run queue của bộ lập lịch cho đến khi trạng thái của nó được thay đổi bởi một sự kiện nào đó. <br/>
-Một process đang ở trạng thái <i>sleep</i> sẽ không được lập lịch trong CPU.<br/><br/>
Để một đoạn code có thể được đưa vào trạng thái <i>sleep</i> thì nó cần thỏa mãn các điều kiện sau đây:<br/>
-Điều kiện 1: Không được đưa vào trạng thái <i>sleep</i> khi đang trong một atomic context. Tức là driver không được <i>sleep</i> khi đang giữ spinlock, seqlock hoặc RCU lock.<br/>
-Điều kiện 2: Không thể <i>sleep</i> khi đang có các disabled interrupt.<br/>
-Điều kiện 3: Sleep trong semaphore là được phép, nhưng cần phải cẩn thận. Nếu một đoạn code sleep trong khi nó đang giữ semaphore thì thread đang đợi semaphore đó cũng sẽ bị đưa vào trạng thái sleep, do đó cần đảm bảo rằng bạn không block luôn kthread sẽ đánh thức sleeping process. (Phần này copy trong sách ldd3, tuy nhiên semaphore đã bị xóa khỏi kernel từ lâu rồi nên không cần quan tâm).<br/>
-Điều kiện 4: Khi một sleeping kthread được đánh thức thì nó không thể biết là nó đã bị loại ra khỏi CPU được bao lâu, và trong thời gian đó, đã xảy ra những thay đổi nào. Do đó, chúng ta không thể tạo ra một giả thuyết nào về trạng thái của hệ thống sau khi đánh thức kthread, mà phải kiểm tra xem những điều kiện cần thiết có đảm bảo không.<br/>
-Điều kiện 5: Chỉ sleep khi chắc chắn rằng có một ai đó sẽ đánh thức bạn trong một thời điểm nào đó. Ngược lại hệ thống sẽ bị hang. Để làm được điều này thì có một yêu cầu nữa là awaker cần phải tìm được bạn để đánh thức.<br/><br/>

## 2 Wait Queue là gì?
Wait queues được sử dụng khi một task có trạng thái RUNNING trong kernel phải đợi một điều kiện nào đó xảy ra để có thể tiếp tục thực thi. Ví dụ như như hàm read() write() ở phần intro đã nói.

Đối với những task này, việc đưa nó vào trạng thái sleep (không làm gì cả) cho đến khi điều kiện (flag) nó chờ chuyển thành true hoặc tài nguyên nó cần có thể sử dụng được. Lúc này ta cần phải được đưa nó về trạng thái thực thi, bằng cách đánh thức nó. Trong kernel, các thao tác này thực hiện bằng <b><span style="color:red">WAIT QUEUE</span></b>

Các task đang có trạng thái là TASK_INTERRUPTIBLE, TASK_UNINTERRUPTIBLE hoặc TASK_KILLALBLE là các task đang nằm trong tình trạng sleep. Các task đang sleep được chia vào 2 loại: interruptible và uninterruptible. Các task uninterruptible là các task không thể đánh thức bằng các signal(tức là bạn không CTRL+C từ userspace app được), những cái này nên hạn chế sử dụng, vì nó có thể sẽ làm treo hệ thống, và bạn không cách nào tắt nó đi được. Một hệ quả thường thấy nếu như bạn dùng uninterruptible không đúng cách là việc máy của bạn không thể reboot được.
b
Trong một hệ thống có thể có nhiều wait queue, chúng được kết nối với nhau trong một linked list. Hơn nữa, mỗi wait queue có thể có một hoặc nhiều task.

Linux kernel dùng một structure tên là <code>wait_queue_head_t</code> để biểu diễn một wait_queue. một queue head có thể được khởi tạo bằng các cách sau:<br/>
-Initialize statically: <code>DECLARE_WAIT_QUEUE_HEAD(name);</code>
-Initialize dynamicly: <br/>
{% highlight c %}
	wait_queue_head_t my_queue;
	init_waitqueue_head(&my_queue);
{% endhighlight %}
<br/>
## 3.Simple Sleep
Một khi đã khởi tạo wait queue, chúng ta có thể sử dụng các hàm được cung cấp để thực hiện việc sleep và wake up các task trong wait queue.
-Linux cung cấp function family: <code>wait_event</code> để thực hiện sleeping:<br>
<code>wait_event(queue, condition)</code><br/>
<code>wait_event_interruptible(queue, condition)</code><br/>
<code>wait_event_timeout(queue,condition,timeout)</code><br/>
<code>wait_event_interruptible_timeout(queue, condition, timeout)</code><br/><br/>
-Sau khi đã sleep, linux cung cấp các hàm sau để đánh thức sleeping process:<br/>
<code>wake_up(wait_queue_head_t *queue)</code><br/>
<code>wake_up_interruptible(wait_queue_head_t *queue)</code><br/><br/>
Hàm <code>wake_up</code> đánh thức tất cả các process trong hàng đợi, phiên bản interruptible đánh thức các process thực hiện một interruptible sleep.<br/>
Thông thường chúng ta sẽ dùng hàm <code>wait_event_interruptible</code>, vì task được sleep bằng hàm này có thể được đánh thức bằng một SIGNAL hoặc một lời gọi đến hàm <code>wake_up</code>, nên việc kiểm tra xem có phải nó được đánh thức bằng một SIGNAL hay không, để đưa ra các phép xử lý đúng là tương đối cần thiết, việc này có thể được thực hiện bằng hàm <code>signal_pending</code>. Ngoài ra nếu bạn không muốn task của bạn sleep quá lâu, thì bạn có thể sử dụng các hàm wait có timeout ở trên.
Nếu bạn đọc kernel code thì bạn sẽ thấy là hàm wait sẽ thực hiện 2 việc: thông báo cho trình lập lịch thực hiện lập lịch một task mới và việc thứ hai là chạy một vòng lặp <i>for</i> để kiểm tra điều kiện nó đang đợi.

Nếu như trong wait queue có nhiều task, hàm <code>wake_up</code> sẽ đánh thức tất cả các task có trong wait_queue, và nó không đảm bảo thứ tự của việc đánh thức này. Điều này khá bất tiện trọng một số trường hợp, chẳng hạn như nếu bạn có một số task đang chờ đợi cùng một resource, khi resource này thỏa mãn thì tất cả các task trong wait queue đều được đánh thức, tuy nhiên resource này chỉ cho phép một task sử dụng tại một thời điểm, thì việc đánh thức này rõ ràng tỏ ra không hiệu quả và có thể dẫn đến lỗi. Để giải quyết trường hợp này, kernel cung cấp các hàm sau:
{% highlight c %}
wait_event_interruptible_exclusive(wait_queue_head_t wq, int condition);
void wake_up_all(wait_queue_head_t *wq);
void wake_up_interruptible_all(wait_queue_head_t *wq);
void wake_up_nr(wait_queue_head_t *wq, int nr);
void wake_up_sync_nr(wait_queue_head_t *wq, int nr);
void wake_up_interruptible_nr(wait_queue_head_t *wq, int nr);
void wake_up_interruptible_sync_nr(wait_queue_head_t *wq, int nr);
{% endhighlight %}

Ở đây <code>nr</code> là số lượng task sẽ được đánh thức.(thường là 1).
Thông thường thì các hàm wait sẽ kiểm tra điều kiện truyền vào (condition flag) ngay tại thời điểm gọi hàm, nếu điều kiện này không thỏa mãn thì nó mới tạo ra một mục mới trong WAIT QUEUE, ngược lại nó sẽ return ngay lập tức, và đoạn code sau lời gọi hàm wait sẽ được thi ngay mà không phải đợi chờ gì cả. WAIT QUEUE về cơ bản chỉ là một cái danh sách liên kết chứa các con trỏ <code>struct wait_queue_entry*</code>

## 4.Blocking and Nonblocking Operations
Phần này sẽ nói về việc xác định xem khi nào chúng ta sẽ đưa process vào trạng thái sleep?<br/>
-Một số operation trong UNIX yêu cầu rằng không được block nó, kể cả nếu như nó không thể thực hiện một cách hoàn toàn. Ngoài ra cũng có một số thời điểm mà process thông tin cho bạn rằng nó không muốn bị block, kể cả nó có thực hiện I/O hay không. Những nonblocking I/O rõ rằng này được thông tin bởi cờ O_NONBLOCK trong <code>filp->f_flags</code>.<br/><br/>
Ở trường hợp blocking operation, đây là mặc định, các thao tác sau đây nên được impelement:<br/>
-Nếu một process gọi hàm <code>read</code> nhưng dữ liệu chưa khả dụng, thì process block. Process được đánh thức ngay khi dữ liệu đến, và dữ liệu được trả về cho người gọi hàm, kể cả nếu lượng dữ liệu trả về ít hơn lượng dữ liệu được yêu cầu (trong argument <code>count</code>).<br/>
-Nếu một process gọi hàm <code>write</code> và buffer đầy, process phải block, và nó phải nawmgf ở một wait queue khác so với wait queue đang được sử dụng cho việc reading. Khi một số dữ liệu được ghi vào hardware device, và buffer bắt đầu có không gian trống, process sẽ được đánh thức và <code>write</code> được gọi thành công, mặc dù dữ liệu có thể chỉ được ghi một nửa so với lượng dữ liệu được yêu cầu.<br/><br/>
Trường hợp O_NONBLOCK flag được set, nonblocking operations nó sẽ return ngay lập tức, cho phép application lấy dữ liệu. Application phải cẩn thận khi sử dụng các <code>stdio</code> function khi đang sử lý các nonblocking files. Cần phải check <code>errno</code>.<br/>
Một cách tự nhiên, O_NONBLOCK rất có ý nghĩa đối với <code>open</code>. Điều này diễn ra khi lời gọi có thể bị block một thời gian dài, ví dụ, khi mở một FIFO mà tạm thời nó chưa có writer, hoặc khi truy cập một disk file với pending lock. Thông thường, việc mở một device có thể thành công hoặc thất bại, mà không cần phải đợi các event bên ngoài. Tuy nhiên, đôi khi việc mở một device yêu cần một thời gian lâu hơn, và chúng ta có thể chọn sử dụng O_NONBLOCK trong hàm <code>open</code> bằng cách trả về lỗi -EAGAIN ngay lập tức nếu như cờ block được set, sau khi bắt đầu tiến trình khởi tạo device.<br/>

## 5.Example
Sau đây mình sẽ trình bày một ví dụ sleep đơn giản.
Đầu tiên tạo file source code <code>oni_sleep.c</code>. Tiếp theo là include các header cần thiết: <br/>
{% highlight c %}
#include <linux/module.h>
#include <linux/uaccess.h>
{% endhighlight %}

Tiếp theo chúng ta define các hằng số cần thiết cho việc xác định device number:<br/>
{% highlight c %}
#define MINOR_FIRST 0
#define MINOR_COUNT 1
{% endhighlight %}

Khai báo các biến cần thiết cho device driver <br/>
{% highlight c %}
static struct cdev oni_sleep_cdev;
static struct class *oni_sleep_class;
static struct device *oni_sleep_device;
static dev_t device_number;
static int sleep_flag=0;
{% endhighlight %}

Để thực hiện việc sleep, cần phải khai báo một head queue, ở đây mình khai báo head queue bằng cách dùng DECLARE (Statically)
{% highlight c %}
DECLARE_WAIT_QUEUE_HEAD(oni_wait_queue);
{% endhighlight %}

Ở đây ngoài các cấu trúc quen thuộc với một device driver, mình có khai báo thêm biến sleep_flag, biến này sẽ được dùng để làm điểu kiện wake up cho process đã bị sleep trước đó (sleeping process sẽ được đánh thức khi sleep_flag=1)<br/>
Tiếp đến là định nghĩa 2 function của file_operations: open() và release()
{% highlight c %}
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
{% endhighlight %}
Tạm thời 2 function này sẽ không làm gì cả <br/>
Kế tiếp là function read. Ở đây hàm read sẽ không thực hiện đọc dữ liệu hay làm gì khác cả. Nó chỉ ghi ra các log để trace quá trình sleep và wakeup mà thôi. <br/>
{% highlight c %}
ssize_t sleep_read(struct file *filp, char __user *buf, size_t count, loff_t *pos)
{
	printk(KERN_INFO "[READ] Enter read function\n");
	printk(KERN_WARNING "[READ] About to sleep\n");
	wait_event_interruptible(oni_wait_queue, sleep_flag == 1);
	printk(KERN_INFO "[READ] read woken up\n");
	sleep_flag = 0;
	return 0;
}
{% endhighlight %}
Ở hàm read, sau khi in ra 2 log line (dmesg), mình đã block hàm <code>read</code>, và chờ cho điều kiện <code>sleep_flag==1</code> thì sẽ đánh thức nó dậy. Mình sử dụng head queue đã khai báo ở đầu, đưa process vào wait_interruptible, tức là người dùng có thể Ctrl+C để thoát user-app khi nó đang đợi <code>read</code> trả về.<br/>
Bây giờ, mình sẽ wake up hàm <code>read</code> từ hàm <code>write</code>, hiện tại hàm write cũng chỉ được dùng để đánh thức hàm read từ hàng đợi mà không có ghi dữ liệu gì sất. Mình sẽ dùng <code>wake_up</code> để đánh thức tất cả các entry trung head queue dậy.
{% highlight c %}
ssize_t sleep_write(struct file *filp, const char __user *buf, size_t count, loff_t *pos)
{
	printk(KERN_INFO "[WRITE] Enter write function\n");
	printk(KERN_INFO "[WRITE] Wake read up ZZZZZZ\n");
	wake_up(&oni_wait_queue);
	printk(KERN_INFO "[WRITE] Exit write\n");
}
{% endhighlight %}

Chúng ta đã có đầy đủ các file operation cần thiết, bây giờ chúng ta có thể khai báo struct file_operations cho oni_sleep
{% highlight c %}
static struct file_operations oni_fops={
	.open = sleep_open,
	.release = sleep_release,
	.read = sleep_read,
	.write = sleep_write,
	.owner = THIS_MODULE	
};
{% endhighlight %}

Bây giờ chúng ta đã có đủ các thành phần để bắt đầu khởi tạo module, hãy bắt tay vào việc:
{% highlight c %}
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
{% endhighlight %}

Tiếp theo tất nhiên là hàm exit 
{% highlight c %}
static void __exit oni_sleep_exit(void)
{
	device_destroy(oni_sleep_class, device_number);
	class_destroy(oni_sleep_class);
	cdev_del(&oni_sleep_cdev);
	unregister_chrdev_region(device_number, MINOR_COUNT);
	printk(KERN_INFO "Say goodbyte to your hand!\n");
}
{% endhighlight %}

Những thứ râu ria khác, nhưng rất cần thiết
{% highlight c %}
MODULE_LICENSE("GPL");            ///< The license type -- this affects available functionality
MODULE_AUTHOR("Oni Ranger");    ///< The author -- visible when you use modinfo
MODULE_DESCRIPTION("A simple Linux char driver for explain sleeping");  ///< The description -- see modinfo
MODULE_VERSION("0.1");            ///< A version number to inform users
module_init(oni_sleep_init);
module_exit(oni_sleep_exit);
{% endhighlight %}

Tiếp theo chúng ta cần tạo ra một user-app để tương tác với device driver đã tạo tên là test.c
{% highlight c %}
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
{% endhighlight %}

Tiếp theo cần một Makefile để compile những source code dã mổ cò
{% highlight make %}
obj-m+=oni_sleep.o
all:
 make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) modules
 $(CC) test.c -o test
clean:
 make -C /lib/modules/$(shell uname -r)/build/ M=$(PWD) clean
 rm test
{% endhighlight %}

<br/>Bây giờ hãy thử xem nó hoạt động như thế nào.<br/>
Sau khi đã compile, chúng ta nhận được file oni_sleep.ko và test. Việc tiếp theo là insert oni_sleep: <code>sudo insmod oni_sleep.ko</code><br/>
Mở một tab mới với thực hiện việc đọc từ device với command: <code>sudo ./test</code><br/>
Ở tab này bạn sẽ nhìn thấy output như sau:
{% highlight c %}
Press anykey to block your read function
....
{% endhighlight %}
Sau khi bạn gõ Enter, tab này sẽ bị block tại đây thay vì return. Mở một tab khác với thực hiện việc ghi vào device với command: <code>sudo ./test write</code>. Như đã đề cập ở trên, hàm write sẽ unblock hàm read.<br/>
Bây giờ quay lại tab read, gõ Enter xem process còn bị block hay không? Wth, hàm read vẫn bị block @@. Tất nhiên là thế rồi, vì mặc dù mình đã gọi hàm <code>wake_up</code> để đánh thức hàm read, nhưng điều kiện <code>sleep_flag==1</code> vẫn chưa được thảo mãn nên nó vẫn tiếp tục bị block. Gõ Ctrl+C để kill nó.<br/>
Bây giờ, mình sẽ set cờ sleep_flag bằng 1 trước khi gọi wake_up, thêm dòng code vào hàm <code>sleep_write</code> như sau:<br/>
{% highlight c %}
......
sleep_flag = 1;
wake_up(&oni_wait_queue);
.............
{% endhighlight %}
Bây giờ compile lại, sau đấy insmod và thực hiện test như cũ. Bạn sẽ thấy tab read được unblock sau khi thực hiện <code>sudo ./test write</code>.<br/>
Bây giờ kiểm tra dmesg:<br/>
{% highlight c %}
[READ] Enter read function
[READ] About to sleep
[WRITE] Enter write function
[WRITE] Wake read up ZZZZZZ
[WRITE] Exit write
[READ] read woken up
{% endhighlight %}

Bây giờ, thử xem <code>wait_event</code> khác <code>wait_event_interruptible</code> như thế nào. Thay <i>wait_event_interruptible</i> trong hàm sleep_write thành <i>wait_event</i>, các tham số vẫn dữ nguyên như cũ, compile lại module, và insert nó vào hệ thống. Tiếp theo, thực hiện test bằng command <code>sudo ./test</code>. Bây giờ, user-app sẽ bị block ở đây, bạn ấn tổ hợp Ctrl+C xem có gì xảy ra? @@ không thể kill chương trình được. Đúng vậy, <code>wait_event</code> sẽ không cho phép chúng ta ngắt process đang bị block, khác với <code>wait_event_interruptible</code>.<br/>
Các hàm <i>wait timeout</i> sẽ return sau một khoảng thời gian(timeout), đơn vị tính là Jiffies 