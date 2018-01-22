# Input/Output Control

Hầu hết các driver cần có khả năng đọc và ghi device - tức là thực hiện các hardware control khác nhau thông quan device driver. Đa phần các device có thể thực hiện các thao tác phức tạp hơn việc chỉ truyền dữ liệu, chẳng hạn như report lỗi, thay đổi baudrate, tự hủy, etc... Những thao tác này thông thường được support bằng command <code>ioctl</code>.

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
<pre>
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
</pre>
Ở file header, mình đã định nghĩa ra một kiểu mới tên là birthday, là một struct gồm 3 interger number. Đây cũng là argument truyền vào cho các lời gọi ioctl ở phần sau. Sau đấy là 3 <b>command number</b> được sử dụng bởi Oni Ioctl. <br/>
Tất cả các cmd number đều sử dụng chung một magic number là 'o' (nó sẽ tự đổi ra int), trong lý thuyết thì các cmd number của cùng 1 device không bắt buộc phải có magic number giống nhau, nhưng trên thực tế, việc sử dụng 1 magic number duy nhất sẽ giúp code dễ quản lý, đẹp mắt, ảo lòi hơn. <br/>
Các cmd number có sequence number lần lượt là 1, 2, 3. Ở cmd number đầu tiên, chúng ta khai báo rằng nó sẽ đọc dữ liệu từ device và sử dụng tham số có kiểu birthday. Ở cmd number thứ 2, chúng ta k dùng tham số. (mấy cái này tượng trưng thôi, có dùng IO hết cũng chả chết, nhưng mà code cleaning is good).<br/>
File header này sẽ được include ở cả ldd và user-space app. <br/><br/><br/>
Tiếp theo sẽ là file source cho ldd, mình tạo 1 file mới tên là <code>oni_ioctl.c</code><br/>
Đầu tiên phải include những header cần thiết vào
<pre>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/version.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/error.h>
#include <asm/uaccess.h>

#include "query_ioctl.h"
</pre>
Tiếp theo chúng ta sẽ define 2 macro cho minor<br/>
<code>#define FIRST_MINOR 0</code><br/>
<code>#define MINOR_CNT 1</code><br/>
ở đây chúng ta sẽ tạo 1 minor duy nhất bắt đầu từ 0.<br/><br/>
Tiếp theo là khai báo một biến dev_t: biến này sẽ lưu device number cho device sắp tới<br/>
<code>static dev_t dev;</code><br/><br/>
Struct cdev (character device) cho device.<br/>
<code>static struct cdev c_dev;</code><br/><br/>
Struct class dùng để tạo ra device file trong thư mục /dev/<br/>
<code>static struct class *c1;</code><br/><br/>
Đây là các giá trị mặc định ban đầu của day, month, year.<br/>
<code>static int day = 11, month = 02, year = 1993;</code><br/><br/>

Bây giờ đến file_operations, trước hết chúng ta cần declare các hàm sẽ được refer từ file_operations.
<pre>
static int my_open(struct inode *i, struct file *f);
static int my_close(struct inode *i, struct file *f);
#if (LINUX_VERSION_CODE < KERNEL_VERSION(2,6,35))
static int my_ioctl(struct inode *i, struct file *f, unsigned int cmnd, unsigned long arg);
#else
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg);
#endif
</pre>
Hàm my_open và my_close không cần thực hiện tác vụ gì cả, nên chúng ta chỉ cần định nghĩa chúng là các hàm có thân hàm rỗng là được.<br/>
Hàm my_ioctl có có prototype khác nhau, phục thuộc vào kernel đang sử dụng.<br/>
Khi đã có các function declaration rồi thì chúng ta có thể khai báo struct file_operations như sau:<br/>
<pre>
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
</pre>






