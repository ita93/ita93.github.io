---
layout: post

category: Linux device driver

comments: true
---
# Input/Output Control
Như đã biết thì Linux OS chia bộ nhớ thành 2 phần riêng biệt là user space và kernel space. Kernel space dùng để thực thi kernel, các extensions của nó và hầu hết các device driver. Ngược lại Userspace là vùng nhớ mà tất cả các ứng dụng thường người làm việc, tất nhiên là tồn tại nhu cầu để giao tiếp giữa 2 phần này với nhau. Trong Linux chúng ta có thể dùng một số phương pháp phục vụ mục đích này như: ioctl, procfs, sysfs,... Trong bài này diễn viên chính sẽ là ioctl.

ioctl = Input and Output Control, được sử dụng để userspace giao tiếp với device driver. Phần lớn ioctl được sử dụng trong các trường hợp mà một số thao tác đặc thù của một device không được hỗ trợ bởi một systemcall mặc định.

Ở user space, linux cung cấp ioctl system call có prototype như sau:<br/>
<code>int ioctl(int fd, unsigned long cmd, ...);</code></br>
Trong C, <code>...</code> là va_arg (một lượng tham số truyền vào không biết trước), tuy nhiên, một system call không thể nào có va_arg được(Tại sao?). Bởi thế, <code>...</code> ở đây là một argument tùy chọn (có thể có hoặc không). argument này có thể là bất kỳ type nào.<br/>

-Prototype của ioctl trong ldd:<br/>
<code>int (*ioctl)(struct inode *inode, struct file *filp, unsigned int cmd, unsigned long arg);</code><br/>
<i>[kernel]inode+filp = [user-space]fd</i></br>
(need a image here)<br/>
-Nếu ioctl có additional arg thì trong kernel-space, nó luôn được truyền như là một biến unsigned long, bất kể biến truyền vào ở user-space là integer hay pointer. Nếu user-space program không pass optional arg thì biến <code>arg</code> trong ldd sẽ là <b>undifined</b><br/>
<i>type checking for ioctl is disabled</i><br/>
- Đối với từng <code>cmd</code> thì có một tác vụ tương ứng được thực hiện<br/>
- Thông thường ioctl sẽ sử dụng <code>switch(cmd)</code> để thực hiện nhiệm vụ của nó.<br/>
<br/><br/>

## 1. Command number (cmd arg) và cách chọn cmd arg.
-Không nên sử dụng cách chọn một set các số bắt đầu từ 0(hoặc 1) để dụng cho cmd arg. Lý do là các <code>ioctl</code> của các driver khác nhau nên sử dụng các cmd arg khác nhau, hay nói cách khác cmd arg là unique value trong toàn hệ thống.<br/>
-Tại sao lại dùng unique value cho toàn bộ cmd arg? Vì khi đấy nếu user-space prog truyền cmd của driver A và ioctl của driver B thì prog sẽ nhận được giá trị trả về là EINVAL thay vì một thao tác không đoán trước được và không đúng mong muốn của người dùng.<br/><br/><br/>
Vậy chọn giá trị nào cho <code>cmd</code>? <br/>
Đầu tiên cần kiểm tra <code>include/asm/ioctl.h</code> và <i>Documentation/ioctl-number.txt</i> để xem những giá trị đã được xí trước để tránh dùng những giá trị này.<br/>

Command numbers (<code>cmd</code>) được định nghĩa bằng cách sử dụng 4 bitfields: type, number, direction, type. Mỗi bitfield có ý nghĩa như sau:<br/>
<code>type</code>: Magic number, ứng với một device driver có một magic number duy nhất. Bitfield này có độ rộng là 8 bits(_IOC_TYPEBITS).<br/>
<code>number</code>: Đây là số thứ tự. Có kích thước 8 bít (_IOC_NRBITS).<br/>
<code>direction</code>: Chiều transfer của dữ liệu (đọc/ghi/nothing): _IOC_NON(không truyền data), _IOC_WRITE, _IOC_READ, nếu vừa đọc vừa ghi data thì có thể dùng _IOC_WRITE|_IOC_READ. view point ở đây là từ user-space application. tức là ĐỌC từ device, GHI vào device.<br/>
<code>size</code> Kích thước của data. Bitfield này có độ rộng phụ thuộc và arch, nhưng thường là 13/14 bits. Kích thước cho từng arch cụ thể có thể được tìm thấy bằng macro _IOC_SIZEBITS. Thật ra field này là không bắt buộc, nhưng việc sử dụng nó giúp cho driver có thể phát hiện errors. Vì chỉ có 13/14bits thì nó sẽ không đủ để ghi kích thước data khi data lớn.<br/><br/><br/>

## 2. Setup command number với asm/ioctl.h
header này cung cấp các macro giúp chúng ta dễ dàng hơn trong việc setup các command number:
-<code>_IO(type, nr)</code> : Command không nhận argument<br/>
-<code>_IOR(type, nr, datatype)</code>: Command đọc dữ liệu từ device<br/>
-<code>_IOW(type, nr, datatype)</code>: Command ghi dữ liệu vào device<br/>
-<code>_IOWR(type, nr, datatype)</code>: Dữ liệu được truyền ở cả hai chiều.<br/>
Đối với các command này: type và number được truyền như các args, ở đây chúng ta không thấy field size, thật ra, nó sẽ được macro tạo ra bằng cách <code>sizeof(datatype)</code><br/><br/>

Cũng trong header này, các macros dùng decode numbers được định nghĩa như sau:<code>_IOC_DIR(nr), _IOC_TYPE(nr), _IOC_NR(nr), _IOC_SIZE(nr)</code> tương ứng (direction, type, sequence number, data size).<br/>
Ví dụ về ioctl command number definations:<br/>
<code>#define ONI_IOCREAD _IOR(ONI_MAGIC_NR,FIRST_SEQ);</code><br/><br/>

## 3. Giá trị trả về của ioctl
- Nếu command number truyền vào không đúng thì giá trị trả về nên là -EINVAL/-ENOTTY<br/>

## 4. Sử dụng argument trong ioctl
-Argument ở đây có thể là integer number hoặc pointer<br/>
-Nếu argument truyền vào là một pointer thì cần đảm bảo rằng user space address là hợp lệ, nếu không nó có thể gây ra kernel ops... Driver cần kiểm tra tất cả các pointer được truyền vào <br/>

## 5. Ví dụ
"Không gì tốt hơn thực hành" Một cao nhân dấu tên, dấu cả chân đã nói thế. Sau đây sẽ là một ví dụ về ioctl.<br/>
Ví dụ này sẽ gồm 2 thành phần: <br/>
- Device driver : oni_ioctl <br/>
- User-space app: oni_app <br/>
Đầu tiên cần tạo ra 1 file header chứa các biến cần thiết để sử dụng.</br>
<code>oni_ioctl.h</code><br/>
{% highlight c %}
#ifndef ONI_IOCTL_H
#define ONI_IOCTL_H
#include <linux/ioctl.h>

type def struct{
	int day, month, year;
}birthday;

//Define 3 command numbers with type (magic number) is 'o', sequen: 1,2,3. Data type query_arg_t
#define QUERY_GET_VARIABLES _IOR('o',1,birthday *)
#define QUERY_CLR_VARIABLES _IO('o',2)
#define QUERY_SET_VARIABLES	_IOW('o',3,birthday *)
{% endhighlight %}
Ở file header, mình đã định nghĩa ra một kiểu mới tên là birthday, là một struct gồm 3 interger number. Đây cũng là argument truyền vào cho các lời gọi ioctl ở phần sau. Sau đấy là 3 <b>command number</b> được sử dụng bởi Oni Ioctl. <br/>
Tất cả các cmd number đều sử dụng chung một magic number là 'o' (nó sẽ tự đổi ra int), trong lý thuyết thì các cmd number của cùng 1 device không bắt buộc phải có magic number giống nhau, nhưng trên thực tế, việc sử dụng 1 magic number duy nhất sẽ giúp code dễ quản lý, đẹp mắt, ảo lòi hơn. <br/>
Các cmd number có sequence number lần lượt là 1, 2, 3. Ở cmd number đầu tiên, chúng ta khai báo rằng nó sẽ đọc dữ liệu từ device và sử dụng tham số có kiểu birthday. Ở cmd number thứ 2, chúng ta k dùng tham số. (mấy cái này tượng trưng thôi, có dùng IO hết cũng chả chết, nhưng mà code cleaning is good).<br/>
File header này sẽ được include ở cả ldd và user-space app. <br/><br/><br/>
Tiếp theo sẽ là file source cho ldd, mình tạo 1 file mới tên là <code>oni_ioctl.c</code><br/>
Đầu tiên phải include những header cần thiết vào
{% highlight c %}
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/error.h>
#include <asm/uaccess.h>

#include "query_ioctl.h"
{% endhighlight %}
Tiếp theo chúng ta sẽ define 2 macro cho minor<br/>
<code>#define FIRST_MINOR 0</code><br/>
<code>#define MINOR_CNT 1</code><br/>
ở đây chúng ta sẽ tạo 1 minor duy nhất bắt đầu từ 0.<br/><br/>
Tiếp theo là khai báo một biến dev_t: biến này sẽ lưu device number cho device sắp tới<br/>
<code>static dev_t dev;</code><br/><br/>
Struct cdev (character device) cho device.<br/>
<code>static struct cdev c_dev;</code><br/><br/>
Struct class dùng để tạo ra device file trong thư mục /sys/class<br/>
<code>static struct class *c1;</code><br/><br/>
Đây là các giá trị mặc định ban đầu của day, month, year.<br/>
<code>static int day = 11, month = 02, year = 1993;</code><br/><br/>

Bây giờ đến file_operations, trước hết chúng ta cần declare các hàm sẽ được refer từ file_operations.
{% highlight c %}
static int my_open(struct inode *i, struct file *f);
static int my_close(struct inode *i, struct file *f);
#if (LINUX_VERSION_CODE < KERNEL_VERSION(2,6,35))
static int my_ioctl(struct inode *i, struct file *f, unsigned int cmnd, unsigned long arg);
#else
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg);
#endif
{% endhighlight %}
Hàm my_open và my_close không cần thực hiện tác vụ gì cả, nên chúng ta chỉ cần định nghĩa chúng là các hàm có thân hàm rỗng là được.<br/>
Hàm my_ioctl có có prototype khác nhau, phục thuộc vào kernel đang sử dụng.<br/>
Khi đã có các function declaration rồi thì chúng ta có thể khai báo struct file_operations như sau:<br/>
{% highlight c %}
statc struct file_operations oni_fops=
{
	.owner = THIS_MODULE,
	.open = my_open,
	.release = my_close,
#if (LINUX_VERSION_CODE < KERNEL_VERSION(2,5,35))
	.ioctl = my_ioctl
#else
	.unlocked_ioctl = my_ioctl
#endif
};
{% endhighlight %}

Đầu tiên khi ldd được insert vào hệt thống, nó sẽ chạy init đầu tiên, nên mình viết hàm init trước cho nó theo thứ tự ahihi. Trước hết là khai báo 2 local variables để lưu giữ result của 1 số lời gọi hàm trong lúc khởi tạo ldd<br/>
{% highlight c %}
static void __init oni_init(void)
{
	int ret;
	struct device *dev_ret;
}
{% endhighlight %}
Tiếp theo, chúng ta sẽ đăng ký device number cho driver, sử dụng <code>alloc_chrdev_region. </code><br/>
{% highlight c %}
...
if((ret = alloc_chrdev_region(&dev, FIRST_MINOR, MINOR_CNT,"query_ioctl"))<0)
{
	printk(KERN_WARNING "Cannot register device number range");
	return ret;
}
{% endhighlight %}
Ở đây chúng ta đăng ký một device driver với major number được cấp phát động và chỉ một minor number duy nhất (0). Hàm <code>alloc_chrdev_region</code> sẽ trả về giá trị âm nếu việc đăng ký thất bại. Device number tạo ra sẽ được lưu vào variable <code>dev</code><br/><br/>
Tiếp theo là khởi tạo character device với cấu trúc cdev ở trên và file_operations chúng ta đã định nghĩa lúc trước. Sau đó dùng hàm <code>cdev_add</code> để thêm nó device vào system<br/>
{% highlight c %}
...
cdev_init(&c_dev, &query_fops);

if((ret = cdev_add(&c_dev,dev,MINOR_CNT))<0)
{
	printk(KERN_WARNING "Cannot register device to kernel");
	return ret;
}
{% endhighlight %}
<br/>
Tạo class cho device, sau bước này, device sẽ xuất hiện trong /sys/class directory. 
{% highlight c %}
...
if(IS_ERR(c1=class_create(THIS_MODULE,"char")))
{
	cdev_del(&c_dev);
	unegister_chrdev_region(dev, MINOR_CNT);
	printk(KERN_WARNING "Cannot create device class");
	return PTR_ERR(c1);
}
{% endhighlight %}
Nếu việc tạo class thất bại, thì chúng ta sẽ phải unregister device number range và xóa struct c_dev đã được khởi tạo.<br/><br/>
Bước cuối cùng trong việc init là tạo device file<br/>
{% highlight c %}
...
if(IS_ERR(dev_ret = device_create(c1,NULL,dev,NULL,"query")))
{
	class_destroy(c1);
	cdev_del(&c_dev);
	unegister_chrdev_region(dev, MINOR_CNT);
	printk(KERN_WARNING "Cannot create device file");
	return PTR_ERR(dev_ret);
}
printk(KERN_NOTICE "LDD inserted successfully");
return 0;
{% endhighlight %}
<br/><br/><br/>
Hàm tiếp theo chúng ta động tới là hàm exit của module. Hàm này cực kỳ đơn giản, chỉ cần phá hết những gì đã làm trong hàm init là được.<i> open và release cũng là một cái tạo ra và một cái đập phá những thứ được tạo ra, tuy nhiên open/release được gọi trên đơn vị fd, còn init/exit là module</i><br/>
{% highlight c %}
static void __exit oni_exit(void)
{
	device_destroy(c1,dev);
	class_destroy(c1);
	cdev_del(&c_dev);
	unregister_chardev_region(dev, MINOR_CNT);
	return PTR_ERR(dev_ret);
}
{% endhighlight %}
<br/><br/><br/>
Bây giờ đến phần chính của bài viết này: IOCTL. Hàm này không có gì ngoài một lệnh switch =))<br/>
{% highlight c %}
#if (LINUX_VERSION_CODE < KERNEL_VERSION(2,6,35))
static int my_ioctl(struct inode *i, struct file *f, unsigned int cmnd, unsigned long arg)
#else
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
#endif
{
	birthday q;
	switch(cmd)
	{
		case QUERY_GET_VARIABLES:
			q.day = day;
			q.month = month;
			q.year = year;
			if(copy_to_user((query_arg_t *)arg, &q, sizeof(query_arg_t)))
			{
				return -EACCES;
			}
			break;
		case QUERY_CLR_VARIABLES:
			q.day = 1;
			q.month = 1;
			q.year = 1970;
		case QUERY_SET_VARIABLES:
			if(copy_from_user(&q,(query_arg_t *)arg,sizeof(query_arg_t)))
			{
				return -EACCES;
			}
			day = q.day;
			month = q.month;
			year = q.year;
			break;
		default:
			return -EINVAL
	}
	return 0;
}
{% endhighlight %}

Trong hàm my_ioctl, chúng ta sử dụng 1 câu lệnh switch-case để thực hiện các hành động tương ứng với mỗi cmd number đã khai báo ở file header.

Tiếp theo là chương trình (user-space) để sử dụng các ioctl cmd đã khai báo:
{% highlight c %}
//filename: query_app.c
#include <stdio.h>
#include <sys/types.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <sys/ioctl.h>

//Include để sử dụng các khai báo.
#include "query_ioctl.h"

void get_vars(int fd)
{
    query_arg_t q;
    if(ioctl(fd, QUERY_GET_VARIABLES,&q) == -1)
    {
            perror("query_apps ioctl get");
    }
    else
    {
            printf("Day: %d\n", q.day);
            printf("Month: %d\n", q.month);
            printf("Year: %d\n", q.year);
    }
}

void clr_vars(int fd)
{
    if(ioctl(fd, QUERY_CLR_VARIABLES) == -1)
    {
            printf("query_apps ioctl clr\n");
    }
}

void set_vars(int fd)
{
    int v;
    query_arg_t
    printf("Enter Day: \n");
    scanf("%d",&v);
    getchar();
    q.day = v;
    printf("ENTER Month: \n");
    scanf("%d",&v);
    q.month = v;
    getchar();
    printf("ENTER Year: \n");
    scanf("%d",&v);
    getchar();
    q.year
    if(ioctl(fd, QUERY_SET_VARIABLES, &q) == -1)
    {
            perror("query_apps ioctl set");
    }
}

int main(int argc, char *argv[])
{
	//Đây là đường dẫn đến dev file
    char *file_name = "/dev/query";
    int fd;
    enum
    {
            e_get,
            e_clr,
            e_set
    }option;

    /*
    Phần còn lại sẽ kiểm tra số tham số được truyền vào, và dựa vào đấy sẽ thực hiện các lời gọi đến hàm tương ứng
    */

    if(argc == 1)
    {
            option = e_get;
    }
    else if(argc ==2)
    {
            if(strcmp(argv[1], "-g") == 0)
            {
                    option = e_get;
            }
            else if(strcmp(argv[1], "-c") == 0)
            {
                    option = e_clr;
            }
            else if(strcmp(argv[1],"-s") == 0)
            {
                    option = e_set;
            }
            else
            {
                    fprintf(stderr, "Usage %s [-g|-c|-s]\n", argv[0]);
                    return 1;
            }
    }
    else
    {
            fprintf(stderr, "Usage %s [-g|-c|-s]\n",argv[0]);
            return 1;
    }

    //Lấy file descriptor của device file
    fd = open(file_name, O_RDWR);
    if(fd == -1)
    {
            perror("query_apps open");
            return 2;
    }

    switch(option)
    {
            case e_get:
                    get_vars(fd);
                    break;
            case e_clr:
                    clr_vars(fd);
                    break;
            case e_set:
                    set_vars(fd);
                    break;
            default:
                    break;
    }

    close(fd);
    return 0;
}


{% endhighlight %}

Sau khi đã hoàn thành cả ldd và app, chúng ta có thể test nó, đầu tiên cần insert module vào kernel: <code>insmod oni_ioctl.ko</code>
Bây giờ mở terminal và enter command <code>sudo query_app</code>, lúc này, nó sẽ thực hiện gọi đến hàm e_get, tức là thực hiện command number QUERY_GET_VARIABLES của ldd. Kết quả in ra màn hình sẽ là giá trị mặc định đã khai báo trong file oni_ioctl.c:
{% highlight shell %}
Day: 11
Month: 2
Year: 1993
{% endhighlight %}

Bây giờ, sử dụng ioctl để thay đổi giá trị của birthday lưu trong ldd bằng cách sử dụng command: <code>sudo query_app -s</code>, sau đó nhập giá trị bạn muốn vào.
Lúc này, nếu sử dụng <code>sudo query_app</code> hoặc <code>sudo query_app -g</code> để in thì bạn sẽ thấy giá trị mới nhập vào sẽ được in ra màn hình.